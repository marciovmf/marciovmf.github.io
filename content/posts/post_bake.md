---
category: post
date:     2026-04-07
slug:     string_interning_and_data_baking
tags:     programming C Project
template: post
title:    String interning and data Baking
---

In game engines, large games, or any sizable codebase that relies heavily on strings as identifiers, there’s a predictable point where those strings stop being harmless.

Early on, they’re convenient. You pass `player_idle` into a system, `diffuse.png` into a loader, `ui.click` into an event dispatcher. It’s readable, flexible, and it keeps the code simple. Nothing feels wrong.

Then the project grows.

Those same strings start to spread. They get copied across modules, subtly renamed, concatenated at runtime—sometimes normalized, sometimes not. You add asserts to catch mistakes. Then a hash function. Then a cache. Eventually someone suggests turning them into enums, and now you’re halfway into building a data pipeline you never planned.

At that point, the problem isn’t just performance. It’s that strings mean different things depending on where and how they’re used.

Humans use names. `player_jump` is immediately meaningful.  
Machines want something else entirely: integers, contiguous memory, predictable access.

Bridging that gap at runtime always carries cost—hashing, lookups, allocations, validation—but more importantly, it creates multiple points where things can quietly diverge.

The natural solution is to stop doing that work at runtime.

This starts as string interning, but once moved to build time it naturally extends to identifier generation and asset embedding.

After doing this by hand across multiple projects, it stopped making sense to keep doing it manually. So I wrote a tool for it: [bake](https://github.com/marciovmf/stdx/tree/main/demo/bake).


## The Bake Approach

Instead of letting your program interpret strings while it runs, you describe your identifiers once in a small, structured file, and the tool generates a C header that your code consumes directly.

At that point, strings stop being runtime inputs and become build artifacts.


## How the Input Becomes Code

The input file defines the identifiers and how they are constructed.

```ini
guard = GAME_IDS

[animation]
idle = player_idle
run  = player_run

[sound]
click = @assets/ui/click.wav
hover = @@assets/ui/hover.wav
```

Each entry generates a symbol composed from three parts:

```
<guard>_<section>_<key>
```

So:

```ini
[sound]
click = ...
```

becomes:

```c
GAME_IDS_SOUND_CLICK
```

Sections are not just grouping—they are part of the identifier. This gives you namespacing without relying on conventions or manual prefixes.

## Values: Strings and Assets

Entries define what gets embedded in the generated output.

A plain value:

```ini
[animation]
idle = player_idle
```

becomes:

```c
#define GAME_IDS_ANIMATION_IDLE 0u                // sequential ID
#define GAME_IDS_ANIMATION_IDLE_STR "player_idle" // stored value
#define GAME_IDS_ANIMATION_IDLE_LEN 11u           // length in bytes
#define GAME_IDS_ANIMATION_IDLE_HASH 0x97F56434u  // hash of the value
```

The identifier is built from the section and key (`animation.idle` → `ANIMATION_IDLE`), while the value (`player_idle`) is what gets stored.

A file reference is a file path prefixed with `@` or `@@`:

```ini
[sound]
click = @@assets/ui/click.wav
```

The `@` prefix tells the tool to read the file at build time and embed its contents as raw bytes.  
Using `@@` does the same, but ensures a null terminator immediately after the data.

This generates:

```c
#define GAME_IDS_SOUND_CLICK 58u                          // unique ID
#define GAME_IDS_SOUND_CLICK_PATH "assets/ui/click.wav"   // stored value
#define GAME_IDS_SOUND_CLICK_HASH 0x110D2028u             // hash of the path 
#define GAME_IDS_SOUND_CLICK_FILE_SIZE 317u               // file size
#define GAME_IDS_SOUND_CLICK_SIZE 318u                    // file content array size

static const unsigned char GAME_IDS_SOUND_CLICK_BYTES[] =
{
  0x52u, 0x49u, 0x46u, 0x46u, ...
  0x00u // present when using @@ instead of @
};

#define GAME_IDS_SOUND_CLICK_CSTR ((const char*)GAME_IDS_SOUND_CLICK_BYTES)
```

From a single entry, the tool produces a stable identifier and the corresponding data, already embedded and sized, with the original path preserved if needed.

Everything is resolved at build time. There is no parsing, no lookup, and no file I/O related to these identifiers at runtime.


## Using It in code

Once you set up your _bake\_file_ as described above you pass it as argument to the tool along with other options and the name of the header file to be generated with the interned content.
The tool itself is intentionally simple:

```
Command Line Usage
usage:
  bake.exe <input.ini> [-o out.h]

The input INI file describes strings and files to bake into a C header.

Entries are grouped into [sections].
IDs are assigned globally by order of appearance.

Entry forms inside a [section]:
  KEY = "string"      # Intern a string (quotes optional)
  KEY = @path         # Embed file bytes (path is relative to the .ini file)
  KEY = @@path        # Embed file bytes and append a null terminator. (path is relative to the .ini file)

```

After generating the header, just include in and use string IDs instead of strigns whenever you want.
Instead of:

```c
load_asset("assets/ui/click.wav");
```

you write:

```c
load_asset(GAME_IDS_SOUND_CLICK_PATH);
```

If you need the file contents or extra information, don't forget the additional
macros that get generated for every entry as discussed previously.:

```c
upload_sound(
    GAME_IDS_SOUND_CLICK,
    GAME_IDS_SOUND_CLICK_BYTES,
    GAME_IDS_SOUND_CLICK_FILE_SIZE
);
```

There is no lookup step. The identifier, the data, and the metadata are already resolved.


## What You Get From This

The immediate effect is that all identifiers become compile-time constants.
Mistakes stop being runtime problems. If you remove or rename an entry in the INI, every use site breaks at compile time.
There is no longer a distinction between “name” and “resolved data”. By the time your program runs, that translation has already happened.

Access becomes direct. An identifier is just an integer, and the associated data is just memory.
Performance improves, but only as a consequence of removing that entire class of work.


## Tradeoffs

This only works if your data is known at build time. You cannot construct new identifiers dynamically, and you cannot load arbitrary user-provided names through this system. If your program depends on that, you still need a runtime path. Embedding files increases binary size, and changing the input requires recompilation. Identifiers depend on input order unless you explicitly stabilize them.


## Build Integration

String interning and asset baking should happen in during build phase. Whenever the input file changes, regenerate the header before compiling. Once that is in place, there is no runtime initialization or registration step. The program starts with everything already defined.

In my [C code documentation tool](doxter_no_nonsense_c_documentation_generator.html), I used **bake** to embed HTML templates at build time, eliminating external file dependencies and allowing the program to be distributed as a single executable. Here’s how it integrates with the project’s CMake rules:

```
# We use the demo/bake tool to bake templates, fonts and strings 
set(BAKE_EXE tools/bake.exe)
set(BAKE_TEMPLATE_INPUT src/doxter_template.bake)
set(BAKE_TEMPLATE_OUTPUT src/generated/doxter_template.h)

add_custom_command(
  OUTPUT ${BAKE_TEMPLATE_OUTPUT}
  COMMAND ${BAKE_EXE} ${BAKE_TEMPLATE_INPUT} -o ${BAKE_TEMPLATE_OUTPUT}
  DEPENDS ${BAKE_TEMPLATE_INPUT}
  COMMENT "[BAKE] templates: ${BAKE_TEMPLATE_INPUT} -> ${BAKE_TEMPLATE_OUTPUT}" 
  VERBATIM
)
```

Nothing special here. Just be sure the tool runs whenever the .bake file changes.

## Closing

There isn’t much else to it. Once you see the problem clearly, the direction becomes obvious. Having a tool for it is a useful addition to the toolbox.
        
