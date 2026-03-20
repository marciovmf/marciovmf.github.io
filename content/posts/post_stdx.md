---
title:    STDX - My Standard C Library
template: post
tags:     programming C project
date:     2026-03-11
category: post
slug:     stdx_my_standard_c_library
---


About two years ago I decided that I would use **C instead of C++ for my personal projects**. Life is too short to program C++ at work *and* in personal projects.

Because of that decision I quickly found myself reimplementing the same basic functionality over and over again: arenas, hash tables, filesystem utilities, string helpers, and similar infrastructure.
Eventually I decided to stop rewriting these pieces in every project and instead implement them once in a reusable way.
At the same time I gathered other bits of common code I had written over the years — some of it originally written in C++, later rewritten in C.
The result became **my own standard library** called [STDX](https://github.com/marciovmf/stdx) .

---

## What is STDX

STDX is a collection of small utility libraries for C that provide implementations of common data structures and system helpers.
Each module solves a specific problem and can be used independently.

All modules are **single-header libraries**.  
The implementation is enabled by defining a macro in exactly one translation unit:

```
#define X_IMPL_ARRAY
#include "stdx_array.h"
```

This allows the library to be dropped directly into a project without requiring a specific build system.

---

## Design Philosophy

The STDX headers follow a very recognizable structure:

- single-header libraries
- C99
- STB-style toggled implementation
- explicit allocation hooks
- minimal dependencies
- straightforward, type-centric APIs

There are no compiler tricks, no heavy metaprogramming, and very little hidden behavior.
The goal is predictable cost and straightforward correctness. The code should be readable from top to bottom and understandable without surprises.

---

## Feature Overview

STDX ships as a set of modules inside src/, each focused, readable, and self-contained.

### Core
- **stdx_common**
  - Portability macros, compiler/OS detection, assertions, bit utilities, typedefs, and tagged result pointers.
- **stdx_time**
  - Cross-platform timers, high-resolution time measurement, time arithmetic, and sleep helpers.
- **stdx_cpuid**
  - Cross-platform CPU information and feature detection.

### Memory & Containers

- **stdx_arena**
  - Bump allocator with chunk growth, mark/rewind support, trimming, and zero-fill helpers.

- **stdx_array**
  - Generic dynamic array with automatic growth and stack-like push/pop operations.

- **stdx_hashtable**
  - Generic hash map with open addressing, tombstones, and user-provided callbacks.


### Strings & Text Utilities

- **stdx_string**
  - UTF-8 utilities, slices, stack-allocated small strings, searching, trimming, conversions, and comparisons.

- **stdx_strbuilder**
  - Fast dynamic string builders for UTF-8 and wide strings.

- **stdx_ini**
  - Minimal INI parser with string interning and flat array storage.


### Platform & System Helpers

- **stdx_filesystem**
  - Path utilities, directory walking, file operations, metadata queries, symlinks, and file watchers.

- **stdx_network**
  - Unified socket API supporting TCP/UDP, IPv4/IPv6, polling, DNS, and multicast/broadcast.

- **stdx_io**
  - Thin wrapper around FILE* for consistent I/O and helpers for reading and writing whole files.

- **stdx_thread**
  - Portable threads, mutexes, condition variables, sleep/yield helpers, and a simple thread pool.


### Diagnostics & Tooling

- **stdx_log**
  - Structured, colorful logging with configurable outputs and log levels.

- **stdx_test**
  - Small testing framework with timing, assertions, and crash reporting.

---

## Tools


In addition to the libraries, the project also includes a small set of command-line tools built on top of STDX.
They are small utilities that solve specific problems and are implemented using the same principles as the library itself: simple code, minimal dependencies, and predictable behavior.

Two examples are currently included in the repository.

### bake

[bake](https://github.com/marciovmf/stdx/tree/main/demo/bake>) is a small utility that helps embed files into C source code.

It reads an input file and generates C code that contains the file's contents as static data. This makes it possible to compile resources such as shaders, configuration files, or other assets directly into a program without relying on external files at runtime.
**bake** is also used to generate **string intern tables**.

String interning is a technique where identical strings are stored only once in memory and referenced through pointers or IDs. Instead of comparing strings character-by-character, the program can simply compare pointers or integer identifiers.

For example, instead of repeatedly allocating and comparing strings like:
```
 "open"
 "close"
 "open"
 "close"
```

the system stores each unique string only once:
```
  open  -> id 1
  close -> id 2
```

Every occurrence of **"open"** then references the same interned entry.

This has several advantages: faster comparisons (pointer or integer comparison instead of **strcmp**), reduced memory usage, stable identifiers for strings used as keys or symbols
Using **bake**, these intern tables can be generated ahead of time and compiled directly into the program.

### doxter

[doxter](https://github.com/marciovmf/stdx/tree/main/demo/doxter) is the documentation generator used for STDX itself.

It parses specially formatted comments inside the headers and produces the documentation available [here]("https://handmadegame.dev/stdx") .
The tool was designed specifically for this project, with the goal of keeping the documentation process simple and tightly integrated with the code.
Like the rest of the project, it avoids heavy dependencies and focuses on straightforward processing of the source files.

---
