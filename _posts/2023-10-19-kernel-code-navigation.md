---
title: "neovim + clangd for kernel development"
date: "2023-10-19 05:58PM"
categories: ["Kernel source code navigation"]
tags: ["neovim", "clangd", "kernel navigation", "IDE", "Language server"]
---

## What's LSP ?

The `Language Server Protocol (LSP)` is an open, `JSON-RPC-based` protocol for use
between source code editors or integrated development environments (IDEs) and
servers that provide "language intelligence tools". Programming language-specific
features like `code completion`, `syntax highlighting` and `marking of warnings`
and errors, as well as refactoring routines. The goal of the protocol is to allow
programming language support to be implemented and distributed independently of
any given editor or IDE. In the early 2020s LSP quickly became a "norm" for
language intelligence tools providers.

The LSP was created by Microsoft to define a common language for programming
language analyzers to speak. Today, several companies have come together to support
its growth, including Codenvy, Red Hat, and Sourcegraph, and the protocol is
becoming supported by a rapidly growing list of editor and language communities.

## What's clangd ?

[`clangd`](https://clangd.llvm.org) is a language server that can work with many
editors via a plugin. clangd understands the C++ code and adds smart features to
editor like `code completion`,`compile errors`, `go-to-definition` and more.

## Clangd features

Here is what clangd can do:

- `clangd` runs the clang compiler on the code as you type, and shows diagnostics
   of errors and warnings in-place.
- The compiler can suggest fixes for many common problems automatically, and
  clangd can update the code for you.
- clangd embeds [`clang-tidy`](https://clang.llvm.org/extra/clang-tidy/) which
  provides extra hints about code problems: bug-prone patterns, performance traps,
  and style issues.
- clangd will index the whole project and can suggest cross references & find the
  references for selected API's.
- Hover over a symbol to see more information about it, such as its type,
  documentation, and definition.
- clangd embeds clang-format, which can reformat your code: fixing indentation,
  breaking lines, and reflowing comments.

## Whats neovim ?

Neovim is a refactor, and sometimes redactor, in the tradition of Vim (which
itself derives from Stevie). It is not a rewrite but a continuation and extension
of Vim. Many clones and derivatives exist, some very clever—but none are Vim.
Neovim is built for users who want the good parts of Vim, and more.

## Installing clangd on Ubuntu (on arm64)

Here is how you can install clangd on ubuntu:

```bash
sudo apt install clangd
manojeda@manojeda-vm$ which clangd
/usr/bin/clangd
```

## Config files 

Below are the config files that would add clang lsp to neovim