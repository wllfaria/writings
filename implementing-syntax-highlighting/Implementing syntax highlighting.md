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

I've met countless strategies of rendering text to a terminal, but for simplicity, I'll explain one of the most straight forward ones. We can think of a terminal screen as a long list of `cells`, those cells are spots where a character could be rendered and, as long as we know the size in rows and columns of this terminal screen, we can calculate where each cell index should be located.

With this simple way of storing our code, we could implement a view to a text file by filling each cell on this `Viewport` with characters of some source code file. This is probably one of the easiest ways to map contents of a file to places on a terminal screen in a flexible enough way that also allows us to apply styling to each individual cell, which is very important for highlighting text.

Starting here, I'll share a bunch of rust code snippets that [can be found here](https://github.com/wllfaria/tree_sitter_syntax_highlight). Lets look at this `Viewport` structure I've mentioned:
```rust
use crossterm::style::Color;

#[derive(Debug, Default, Clone)]
pub struct Cell {
    symbol: char,
    fg: Color,
    bg: Color,
}

#[derive(Debug)]
pub struct Viewport {
    buffer: Vec<Cell>,
    pub size: (usize, usize),
}
```

This is the simplest form to represent a terminal screen we will use, we can start this Viewport by setting the buffer vector to have a size of `columns * rows` available, and if we pretend we have every cell filled, we can simply calculate where each cell should be displayed before we print to the terminal.

<details>

  <summary>In case you're curious, this is how to calculate where each cell should be displayed</summary>

```rust
for (idx, cell) in self.buffer.enumerate() {}
    let width = self.size.0;
	let column = idx / width;
	let row = idx % width;
}
```
</details>

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