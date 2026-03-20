---
category: post
date:     2025-12-19
slug:     minima_a_tiny_command_language_you_can_embedd_anywhere
tags:     programming C Project
template: post
title:    Minima - A tiny language you can embed anywhere
---

I didn’t set out to build a programming language.
Minima started as a testbed—a way to exercise
[STDX](stdx_my_standard_c_library.html) in something more realistic than isolated unit tests. I needed something that would stress parsing, memory management, data structures, and execution flow… all at once. So instead of writing more tests, I wrote a language.

And then it became useful. That raised a more interesting question.

---

## Why Minima exists

At first, Minima was just a demo interpreter. But it quickly became clear that it solves a real problem:
> How do you embed simple, extensible behavior into a C project without dragging in a full general-purpose scripting runtime?

Most options fall into two extremes: either too heavy (Lua, Python, etc.) or too limited (ad-hoc config formats, JSON, etc.).
Minima sits in the middle: it is a scripting language, but one designed specifically for embedding—small, fully controllable from C, with no hidden semantics and a very simple runtime. It’s intended to be used as a general-purpose embedded DSL.

To understand why that works, you need to look at it was designed **not** to be.

---

## Design influences (and rejections)

Minima didn’t come from a vacuum. It’s shaped as much by what it avoids as by what it embraces.

Lua is the obvious starting point. It’s one of the most popular embedded languages, and for good reason: it’s clean, well-designed, and integrates well with C. However, it’s still a full language runtime. Tables, metatables, coroutines, garbage collection… even when you only need a small DSL, you inherit a lot of machinery.

Minima deliberately avoids that weight:

- no object model  
- no VM complexity beyond what’s needed  
- no features you didn’t ask for  

On the other end of the spectrum is Tcl.

Tcl showed that “everything is a command” is a powerful idea. It enables a uniform and highly extensible model where the language itself is just a thin layer over command dispatch.

However, in practice, Tcl becomes highly ambiguous. It is full of subtle, context-dependent rules where behavior changes depending on how expressions are interpreted. You’re constantly reasoning about whether something is taken literally, treated as a string, or evaluated in a different context—and whether escape sequences will be expanded or not.

The result is a model that is powerful, but difficult to predict.

Minima takes a different approach:

> Keep the core idea—everything is a command—but remove the ambiguity.

This leads to a simpler and more explicit model:

- no implicit substitutions  
- no hidden evaluation phases  
- commands decide explicitly what to evaluate  

---

## The core idea: everything is a command

Minima has one fundamental rule: **everything is a command**. There are no keywords. No special syntax. No hidden constructs. No `if`. No `while`. No `for`. And yet… you can write all of those. Because they’re just commands.

```
if (expr $n > 10) {
  print "large\\n"
} else {
  print "small\\n"
}
```
That looks like a language construct—but it isn’t. It’s just a command called
`if`. This leads to one key property: **The language is completely extensible.**

You don’t extend Minima by modifying the parser.  You extend it by registering new commands. Once you accept that, the rest of the design follows naturally.

---

## Minimal syntax, by design

The entire syntax of the language fits in a few rules:

- A program is a sequence of commands
- Commands are separated by newline or `;`
- First token = command name
- Remaining tokens = arguments
- `{ ... }` = block
- `( ... )` = subcommand (exactly one command)
- `#` = comment

That’s it.

No precedence rules baked into the language.  
No expression grammar (those live in commands like `expr`).  
No special forms.

This simplicity is not a limitation—it’s what makes the model predictable.

---

## Latent behavior: evaluation is explicit

The most important (and subtle) design choice is this:

Arguments are **not** automatically evaluated.

Commands receive **AST nodes**, not values. Each command decides **if**, **when**, and **how** to evaluate its arguments.

This gives you surgical control over behavior. Different commands decide how evaluation happens:

- `set` evaluates the value, but not the variable name  
- `if` evaluates the condition, then may execute a block  
- `list` dispatches subcommands dynamically  

This is what makes Minima feel simple on the surface but powerful and extensible underneath.

It also makes implementing new commands trivial—because you’re not fighting the runtime.

---

## A quick taste

```
set name "Minima"
set n (expr 40 + 2)
print (format "name=${name}, n=${n}\\n")

```
Lists:

```
set xs (list create a b c)
list push $xs d

foreach item $xs {
  print (format "item=${item}\\n")
}
```

Dynamic evaluation:

```
eval "print \\"hello from eval\\n\\""

```
Include files:

```
include "script.mi"
```

All of these are just commands.

At this point, the important question becomes: how does this integrate into a real system?

---

## Embedding: the real goal

Minima is not meant to be a standalone language. It’s meant to be embedded.

The embedding model is straightforward:

1. Create a context  
2. Register built-in commands  
3. Register your own commands  
4. Parse + execute  


```
XArena *arena = x_arena_create(8 * 1024);
MiContext ctx;

if (!arena) {  /* host error handling */ }

if (!mi_context_init(&ctx, arena, stdout, stderr)) {
  /* host error handling */
}

if (!mi_register_builtins(&ctx)) {
  /* host error handling */
}

mi_register_command(&ctx, "mycmd", cmd_my_cmd);

MiParseResult p = mi_parse(arena, source);
if (!p.ok) {
  fprintf(stderr, "%d:%d: %s\\n",
      p.error_line, p.error_column,
      p.error_message ? p.error_message : "parse error");
}

MiExecResult r = mi_exec_block(&ctx, p.root, false);
if (r.signal == MI_SIGNAL_ERROR) {
  fprintf(stderr, "%d:%d: %s\\n",
      ctx.error_line, ctx.error_column,
      ctx.error_message ? ctx.error_message : "runtime error");
}

x_arena_destroy(arena);

```

The project's [README](https://github.com/marciovmf/stdx/blob/main/demo/minima/readme.md) covers this with more details.

---

## Why this works well in C

Writing something like this in C is usually painful. Minima was practical to implement because STDX provides the missing pieces that C doesn’t:

- Arena allocator for fast, simple lifetime management  
- Dynamic arrays for AST and runtime structures  
- Strings and slices for no constant copying  
- Filesystem + IO helpers for `include`, etc.  

Without that, I’d spend a lot of time fighting memory and data structures. With STDX, the implementation was straightforward—and it even exposed bugs and improvement points in STDX itself. That’s the real experiment here:

> Can you build a small, expressive language in C without losing your sanity?

Turns out: yes, if your foundation is right.

---

## Where this is going

Minima is already useful as:

- A scripting layer for tools  
- A configuration language with behavior  
- A DSL for build systems (Minima + Weld)  
- A templating backend (via `eval` + `include`)  
- A foundation for tools like SLAB  

And it’s still tiny. No JIT.  No optimizer.  No complex runtime.  Just commands.

---

## Final thought

Minima is not trying to compete with Lua or Python. It’s trying to answer a simpler question:

> What is the smallest *useful* language you can embed in a C program?

Right now, the answer looks like this:

- One syntax rule: commands  
- One extension mechanism: register functions  
- One evaluation model: explicit  

Everything else is optional.
