---
title: Implementing syntax highligting
draft: true
summary: How editors implement syntax highlighting and why tree sitter is incredible.
date: 2024-07-29T11:35:00-03:00
tags:
  - rust
  - text-editor
  - tree-sitter
  - code
---

## How editors render text
There are many ways editors render text to the screen, some render at certain FPS [(looking at you, zed)](https://github.com/zed-industries/zed). and others render every time an event happens, which is a common way of rendering text on modal text editors, each approach has its own quirks, but as I'm a huge advocate of the command line, I'll be mainly using [event-driven](https://en.wikipedia.org/wiki/Event-driven_architecture) rendering and modal text editors (like vim) as the subject of this article, although most of the things also apply to other implementations.

I've met countless strategies of rendering text, but for simplicity, I'll show here one of the simpler ways I've stumbled upon. We can think of a terminal screen as a long list of `cells`, those cells are spots where a character could be rendered and, as long as we know the size in rows and columns of the terminal, we can calculate where each cell index should be located.

```rust
use crossterm::style::Color;
use crossterm::*;

#[derive(Debug, Clone)]
pub struct Cell {
    symbol: char,
    fg: Color,
    bg: Color,
}

impl Default for Cell {
	fn default() -> Cell {
		Cell {
			symbol: ' ',
			fg: Color::Reset,
			bg: Color::Reset,
		}
	}
}

#[derive(Debug)]
pub struct Viewport {
    buffer: Vec<Cell>,
    pub size: (usize, usize),
}

impl Viewport {
	pub fn new() -> Viewport {
		let (cols, rows) = terminal::size().unwrap();
		Viewport {
			buffer: vec![Default::default(); (cols * rows) as usize],
			size: (cols as usize, rows as usize)
		}
	}
}
```

Here, we create a single vector of cells to be more efficient with the total size of `columns * rows` of the terminal, which we get from [crossterm](https://github.com/crossterm-rs/crossterm), and initialize every cell to our default cell, which is merely a space without any style to it, this style is what we will fill with colors when we implement syntax highlighting later.

This simple structure represents our terminal viewport in a `virtual` space, allowing us to manipulate its content and later print everything to the actual output, but this is useless unless we have a way to fill this viewport with actual text from some source file.

## Storing text in memory
The inner representation of the contents of a text file within a code editor is a whole universe of discussion, there are many data structures and techniques for efficiency and ease of use, some editors may use a [Piece Table](https://en.wikipedia.org/wiki/Piece_table), others might use a [Gap Buffer](https://en.wikipedia.org/wiki/Gap_buffer), but a very common way is to use a [Rope](https://en.wikipedia.org/wiki/Rope_(data_structure)), which is used by some popular editors, like [Helix](https://github.com/helix-editor/helix). Depending on your project scope, sometimes even a simple vector of strings can cut it.

We are not going to go into the pros and cons of each implementation, this might be subject for another article, so let's just pick ropes to be our data structure of choice.

There is a very neat rust library called `ropey` which provides a very nice rope interface, and its actually very easy to use. So we can just define our `TextObject` struct to hold an instance of a rope, which we will use to manipulate the source code text and display it onscreen.
```rust
#[derive(Debug)]
pub struct TextObject {
    content: ropey::Rope,
}

impl TextObject {
    pub fn new(source_code: &'static str) -> TextObject {
        TextObject {
            content: ropey::Rope::from_str(source_code),
        }
    }
}
```

Lets also make a public method to easily get a range of lines to display onscreen, so we can iterate over these lines and display them.
```rust
pub fn get_within(&self, range: Range<usize>) -> impl Iterator<Item = &str> {
	self.content
		.lines_at(range.start)
		.take(range.end - range.start)
		.map(|s| s.as_str().unwrap_or(""))
}
```

This method just takes in a range of lines to get from the source code, and returns an Iterator for each of the lines inner `&str`, this is a handy way to not take too much lines and waste computation on lines we will not render. We are just a built-in function from `ropey`, which allows us to get an iterator that starts at the line we specify on its argument.

We are almost ready to render things onscreen, the missing part is that our `Viewport`doesn't really have a way to fill its cells, let's do that.

## Rendering Text
Lets modify our `Viewport` by adding a few new functions, one will fill the cells of our buffer from the iterator we can get from our `TextObject`. and the other will actually render our buffer to stdout.

The approach we will follow here is a flexible approach that can later be extended to add double buffering, and also other optimizations, but I'll talk about them later, lets start by implementing a function to fill the viewport from the iterator.

```rust
pub fn fill<T, U>(&mut self, mut code: T)
where
	T: Iterator<Item = U>,
	U: AsRef<str>,
{
	for row in 0..self.size.1 {
		let line = code.next();
		for col in 0..self.size.0 {
			let symbol = match line {
				Some(ref l) => l.as_ref().chars().nth(col).unwrap_or(' '),
				None => ' ',
			};
			let symbol = match symbol {
				s if s.is_whitespace() => ' ',
				s => s,
			};
			self.set_cell(col, row, symbol);
		}
	}
}
```

Ok, we are doing quite a lot here, so let me explain, the first piece of this function is its declaration, our function takes in a generic argument that can be any iterator that yields `&str` as its inner type.

Our inner loops go over every possible cell within the Viewport, and checks if there is any available character to store on our buffer, otherwise we just store an empty space. We are calling a utility function that looks like this:

```rust
pub fn set_cell(&mut self, col: usize, row: usize, symbol: char) {
	let pos = row * self.size.0 + col;
	self.buffer[pos] = Cell {
		symbol,
		fg: Color::Reset,
		bg: Color::Reset,
	}
}
```

Let's not worry too much about the styles for now, we will get back to them once we start implementing Syntax highlighting. We are almost done, let's just 
## How editors highlight text
This is another problem that has a lot of different solutions, some better, some worse. for the most part, modern editors use [Tree-Sitter](https://tree-sitter.github.io/tree-sitter/) to highlight text (and much more). Tree Sitter became the most popular way to handle syntax trees for many applications, it is fast, easy to use, and extensible in a way that is actually impressive. Despite that, some very popular editors actually don't use it [(looking at you, VSCode)](https://code.visualstudio.com/api/language-extensions/syntax-highlight-guide#tokenization), favoring the use of Regex or other ways to parse grammars.

### What is tree sitter
Tree sitter is a parser generator that can parse grammars into a concrete syntax tree.

Woah... those are some funny words, magic man. Putting simply, tree sitter allows you to describe how to parse a language into an [CST](https://en.wikipedia.org/wiki/Parse_tree), and give you a lot of ways to traverse and manage this tree.

What makes tree sitter compelling is that it is efficient, and allow you to implement **incremental** parsing, which is a game changer, specially when talking about large files.

## Syntax highlighting with tree sitter
To use tree sitter in basically any way, we need to set some things up, each language you want to parse will need a `Parser` which is the core component that turns the source code into an syntax tree that tree sitter can work onto. After making a parser, we need to tell it the rules by which our language should be parsed, and after that we can throw any piece of code of the given language and it will get parsed.

```rust
use tree_sitter::Parser;

fn main() {
	let source_code = include_str!("main.rs");
	let mut parser = Parser::new();
	let language = tree_sitter_rust::language();
	parser.set_language(&language).unwrap();
	let tree = parser.parse(&source_code, None).unwrap();
}
```

For simplicity sake, I'll not be showing any form of error handling on the examples given here, if you want to see the full version, check it out on the [repository for this article](https://github.com/wllfaria/tree_sitter_syntax_highlight).

When we parse some source code, we will get a tree from tree-sitter, this tree is what enable us to query for captures, which is how we will be calling every token that matches some query rule. Speaking of which, this tree is cool but we need a way to find out where each token is on the file so we can apply highlighting to it.