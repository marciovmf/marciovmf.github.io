---
category: post
date:     2026-02-02
slug:     doxter_no_nonsense_c_documentation_generator
tags:     programming C Project
template: post
title:    Doxter - No nonsense C documentation generator
---

I built **doxter** as a demo application for the STDX library.

The original goal wasn’t to create a documentation tool. The goal was to exercise the library in a real program and see where it breaks, where it feels good, and where it doesn’t. That meant file handling, string processing, command line parsing, HTML generation, theming—everything that usually exposes weak spots.

Still, I didn’t want a throwaway demo. If I was going to write it, it had to be something I could actually use.

So doxter ended up as a small documentation generator for C projects. It scans `.c` and `.h` files, extracts documentation comments, and produces static HTML. No runtime, no dependencies, no layers.

You can find the project here:  
<https://github.com/marciovmf/stdx/tree/main/demo/doxter>

And here is a real example of documentation generated with it (STDX itself):  
<https://handmadegame.dev/stdx/>

---

# What it does

At its core, doxter is extremely simple. It reads source files, looks for documentation comments, associates them with the symbols that follow, and emits static HTML pages. There’s no attempt to deeply understand the code. It just follows a few predictable rules and stays out of the way.

---

# Command line usage

```
Usage:

  doxter [options] input_files

Options

  -f <path-to-doxter.init>         = Path to doxter.ini.
  -d <path-to-output-dir>          = Specify output directory for generated files.
  -p <project-name>                = Project name.
  --skip-static                    = Skip static functions.
  --skip-undocumented              = Skip undocumented symbols.
  --skip-empty-defines             = Skip empty defines.
```

The interface is intentionally minimal. You point it at a configuration file, choose an output directory, pass your input files, and it generates documentation.

---

# Documentation comments

doxter relies on a very straightforward convention. Any comment block written in this format

```
/**
 * Documentation text
 */
```

is attached to the symbol that immediately follows it. The content of that block becomes the documentation in the generated HTML. There’s no custom language or complex parsing involved.

For functions, the comment block supports a few tags. The brief description is optional; if you don’t explicitly mark it, whatever comes right after `/**` is treated as the brief. Parameters can be documented with `@param <name>`, optionally followed by a description, and return values can be described with `@return`.

There is one special case. If a comment block starts at line 1, column 1 and is followed by an empty line, it is treated as a file-level comment. That content is placed at the top of the generated documentation page and works well as an overview.

---

# Configuration

doxter is driven by a single configuration file in INI format. This file defines project metadata, filtering behavior, markdown support, and the visual theme.

Here is a real example taken from the STDX project:

```
[project]
name = stdx
url = http://github.com/marciovmf/stdx
brief = "STDX is a minimalist, modular C99 utility library inspired by the STB style.
It provides a set of dependency-free, single-header components that extend common programming functionality in C"

skip_static_functions = 1
skip_undocumented = 0
skip_empty_defines = 1
markdown_gobal_comments = 1
markdown_index_page = 1

[theme]
color_page_background = ffffff
color_sidebar_background = f8f9fb
color_main_text = 1b1b1b
color_secondary_text = 5f6368
color_highlight = 0067b8
color_light_borders = e1e4e8
color_code_blocks = f6f8fa
color_code_block_border = d0d7de
border_radius = 10

color_tok_pp  = 6f42c1
color_tok_kw  = 005cc5
color_tok_id  = 24292f
color_tok_num = b31d28
color_tok_str = 032f62
color_tok_crt = 032f62
color_tok_pun = 6a737d
color_tok_doc = 22863a

font_code    = "Cascadia Mono"
font_ui      = "Helvetica", "Verdana", "Calibri"
font_heading = "Helvetica"
font_symbol  = "Segoe UI"
```

The configuration keeps everything explicit. There is no hidden behavior, and changing the look or filtering rules is just a matter of editing values.

---

# Typical workflow

Using doxter follows a predictable flow. You start by creating a `doxter.ini` file. Then you add documentation comments directly in your `.c` and `.h` files. Once that is done, you run the tool with your configuration and input files:

```
doxter -f doxter.ini -d docs file1.h file2.c file3.h 
```

After it finishes, you open the generated HTML in a browser. That’s the entire loop.

---

# Why it exists

Most documentation tools for C try to do too much or impose too much structure. doxter deliberately does less. It doesn’t try to interpret your code beyond what is necessary, and it doesn’t force you into a specific way of writing.

It exists because I wanted a real program to validate STDX and because I wanted a documentation tool that matches how I already write C.

---

# Final thoughts

doxter is intentionally small. It reads files, extracts comments, and renders HTML. That’s all it does.

If you already document your code like this:

```
/**
 * Adds two vectors.
 */
vec3 vec3_add(vec3 a, vec3 b);
```

then you already have documentation. doxter just turns it into something readable.
