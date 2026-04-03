---
category: post
date:     2026-04-03
slug:     stdx_retrospective
tags:     programming C Project STDX
template: post
author:   marciovmf
title:    Lessons Learned After One Year Designing a large C library
---

I’ve been a developer for over 20 years, much of that working on AAA games, tools, and engines. At some point I realized: life is too short to write C++ in your 9-to-5 job and then come home and write more of it.

About a year and a half ago, I switched to using only C for my own projects.

That solved one problem and created another. The C standard library is very basic. I kept rewriting the same pieces over and over until I decided to port parts of my old C++ code into small C libraries to move faster.

That’s how [stdx](stdx_my_standard_c_library.html) started.

## It’s never about “the container”

At the beginning it’s easy to think in terms of components: array, hashtable, filesystem, string builder. In practice, those don’t matter much on their own. What matters is how they behave together.

A container that looks fine in isolation but is awkward to use in real code is a bad container.

That’s how stdx_array and stdx_hashtable ended up with the same shape: a small generic core, and a typed layer generated with macros.

The difference shows up at the call site.

Without typed wrappers:

```c
XArray arr;
x_array_init(&arr, sizeof(int));
x_array_push(&arr, &value);
```

With typed wrappers:

```c
XArray_int arr;
x_array_int_init(&arr);
x_array_int_push(&arr, value);
```

The implementation is the same. The usage is not.

The generic layer keeps the implementation manageable. The typed layer is what you actually want to use. Without that split, you either duplicate everything or force users into casts and awkward call sites. Neither holds up.

Most of the time the right question is not “is this implementation clean?” but “what does this feel like to use 200 times a day?”

## Single-header forces discipline

Adding code from a library to a project shouldn’t require build system changes or installation steps. It should just work. You drop the file in, include it, and move on.

A single header makes that possible. However, it demands discipline. You can’t spread things across files, so the structure has to be clear from the start.

Every module ended up looking like this:

```c
#ifndef X_ARRAY_H
#define X_ARRAY_H

/* public API */

#ifdef __cplusplus
extern "C" {
#endif

X_ARRAY_API void x_array_init(XArray* arr, size_t elem_size);

#ifdef __cplusplus
}
#endif

#ifdef X_IMPL_ARRAY

/* implementation */

#endif
#endif
```

That constraint forces consistency. Every module follows the same layout, the same override points, the same compile model.

## Consistency pays off more than perfect APIs

Because STDX is a collection of libraries, consistency across modules matters for ergonomics.

Allocator overrides work the same way everywhere:

```c
#ifndef X_ARRAY_ALLOC
#define X_ARRAY_ALLOC(sz) malloc(sz)
#define X_ARRAY_FREE(p) free(p)
#endif
```

The same pattern shows up in every module. You don’t have to relearn anything or guess how memory works in each part of the library.

That removes friction when switching between modules. After a while, you stop thinking about the library and just use it.

## Minimalism is about removing ambiguity

I have a strong inclination toward minimalism in general, and that carries into code as an urge to solve the actual problems without adding to them.

That mindset shaped how STDX was designed.

“Minimal” APIs often just leave things out. That’s not useful. The goal is not fewer functions, it’s fewer unclear cases.

What matters is removing ambiguity and unnecessary options.

The filesystem module is a good example. Mixing raw strings, slices, and paths everywhere sounds flexible, but it makes ownership and intent unclear.

```c
XFSPath path;
x_fs_path_from_cstr(&path, "assets/textures/wood.png");

XFSPath parent;
x_fs_path_parent(&parent, &path);
```

There is a clear primary type, and most operations work on it.

Conversions still exist, but they’re explicit:

```c
XSlice name = x_fs_path_basename_as_slice(&path);
```

It’s not smaller. It’s easier to reason about.

## Macros are fine if they stay honest

C doesn’t give you generics, so you either duplicate code or use macros.

In stdx, macros generate typed APIs:

```c
X_ARRAY_TYPE(int)
```

Which expands to:

```c
typedef XArray XArray_int;
void x_array_int_push(XArray_int* arr, int value);
```

No new behavior. Just less repetition.

If the expanded code looks like something you’d write by hand, the macro is doing its job.

## Don’t take control away from the user

Most modules follow the same rule: don’t assume too much.

Allocators can be overridden. Implementation is opt-in:

```c
#define X_IMPL_ARRAY
#include "stdx_array.h"
```

Nothing forces a global state or hidden allocation strategy.

The library gives you defaults, but doesn’t lock you into them.

## Performance shows up in structure, not tricks

Most performance decisions in stdx are structural.

Contiguous memory, stable addresses, simple iteration.

For example, array growth is explicit:

```c
x_array_reserve(&arr, 1024);
```

No hidden realloc surprises in tight loops.

Small optimizations that complicate the API are avoided unless there’s a real reason. If the structure is right, the hot paths are already cheap.

## The library changes how you approach problems

One thing that changes is how quickly you reject certain designs. Anything that relies on hidden state, implicit ownership, or unstable memory starts to feel wrong almost immediately. You don’t need to fully reason about it anymore, it just doesn’t fit. That shortens a lot of decisions. Instead of exploring multiple directions, you converge faster because you already know what good looks like in practice. You stop chasing “maybe this could work” and start filtering based on constraints that have already proven themselves.

It also shifts your focus from isolated solutions to integration. You stop designing pieces in isolation and start thinking about how they will be used together. Naming, ownership, lifetime, and data flow are part of the design from the beginning, not something you fix later. That naturally eliminates a lot of accidental complexity. Not because you are trying to simplify things, but because anything that doesn’t compose well gets pushed out early.

## What held up and what didn't

After a year, I think the shape the library took was mostly organic. I avoided being opinionated on anything I couldn’t back up with actual experimentation and measurements

The simple things held up. The single-header approach still pays off. The allocator override pattern was worth the discipline. The typed macro layers are not pretty, but they remove friction where it matters.

New modules don’t require inventing new patterns anymore. They tend to follow the same structure almost automatically. That’s probably the strongest signal that the core ideas are working.

A subtle but dangerous design pitfall I noticed early is consistency turning into constraint. It’s easy to force every module into the same shape just because the rest of the library looks that way. That works until it doesn’t. When that happens, you start forcing solutions instead of letting the problem define them. Some parts of stdx only look simple because they fit the problem well. Forcing the same shape everywhere can make things worse, not better.

Consistency is about predictability in usage, not subjective “cleanliness”. That line is not always obvious.


## In retrospect

Stdx ended up as a set of small, single-header C libraries that cover the pieces I kept rewriting. Each module can be dropped into a project without touching the build system, and they all follow the same structure, naming, and allocation model.

The APIs are shaped around how they are actually used. Containers have a generic core with typed wrappers so the call sites stay clean without duplicating implementations. Memory ownership is explicit, data stays stable, and there’s no hidden state.

It’s not complete and it’s not trying to be. It’s the subset of code where the trade-offs are understood and the design holds up under repeated use.

The current shape of stdx took more iteration than I expected, but that’s why it holds up.
