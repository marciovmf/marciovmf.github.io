---
category: post
date:     2026-03-27
slug:     you_probably_dont_need_a_test_framework
tags:     programming C Project tests
template: post
title:    You Probably Don’t Need a Test Framework
---

There is a quiet kind of satisfaction in removing complexity rather than adding it. Not because simplicity is trendy, but because most of the time, the problem never required the machinery we instinctively reach for. Unit testing in C is one of those areas where people often overcomplicate things. They bring in frameworks, runners, generators, and layers of abstraction… when all they really need is the ability to call functions and observe outcomes. That mismatch has always bothered me.

Good software should not be artificially complex. If the problem is small, the solution should have the courage to stay small. And yet, this is exactly where things tend to spiral. A handful of tests turns into a dependency. The dependency turns into a build concern. The build concern turns into configuration, conventions, and rules that have very little to do with the original problem. At some point, the cost of testing starts competing with the value of the tests themselves.

Stripped down to its essence, testing is not complicated. A test suite is simply a set of function calls and a mechanism to count successes and failures. There is no fundamental requirement for frameworks, runners, or code generation. Those are tools built around the problem, not the problem itself. The core remains the same: execute code, check results, and report what happened.

Out of that realization, I put together a very simple test machinery to handle the tests of the entire STDX library. It does not try to be clever. It does not try to be complete. It does not try to redefine how tests should be written. It simply embraces what C already gives you: functions, return values, and control over execution.

It is small, self-contained, and honest about what it is. A minimal unit test framework built around a tiny runner, a handful of assertion macros, readable output, and just enough crash diagnostics to be useful.

For most simple test needs, that is enough.
And in software, “enough” is a quality worth protecting.

---

## STDX tests

This tiny test library is part of the [STDX library](https://handmadegame.dev/stdx_my_standard_c_library.html) I wrote to become **my** standard library.

The library follows the single-header style. You include `stdx_test.h` wherever you need the declarations, and in one translation unit you define `X_IMPL_TEST` before including it to compile the implementation. Under the hood, the header pulls in the logging and timing modules it depends on. If their implementations are not already compiled elsewhere, it enables them internally. In practice, that means you get a complete test setup from a single include and a single define. No package manager, no external dependency, no build ceremony.

The model is intentionally direct. A test is just a function returning `int32_t`. Zero means success. Non-zero means failure. Test cases are described with a small struct containing a name and a function pointer, and the `X_TEST(name)` macro turns a function into a test entry without noise.

This is a design that respects C instead of trying to abstract it away.

---

## What `stdx_test.h` Actually Does

One of the things I like most about `stdx_test.h` is that there is no illusion here. It is not a giant framework disguised as a header. It is, essentially, one runner function plus a handful of macros wrapped around plain C functions. That is not a criticism. In fact, that is the point. The whole library is small enough that most C programmers could write something similar themselves in an afternoon. `stdx_test.h` just packages that idea into a reusable, consistent form. 

At the type level, the model is almost as small as it can be. A test is just a function pointer returning `int32_t`, and a test case is just a name paired with that function pointer. The `X_TEST(name)` macro is only a convenience that turns a function into a `{ "name", name }` entry. 

```
typedef int32_t (*STDXTestFunction)();

typedef struct
{
  const char *name;
  STDXTestFunction func;
} STDXTestCase;

#define X_TEST(name) {#name, name}

```

That is the registration system. Just an array of structs.

The assertion layer is equally direct. Each `ASSERT_*` macro checks a condition, logs an error through `x_log_error`, and immediately returns `1` from the current test function on failure. `ASSERT_FALSE` is just `ASSERT_TRUE(!(expr))`. `ASSERT_NEQ` is the same idea inverted. `ASSERT_FLOAT_EQ` does one extra thing: it compares with a fixed epsilon, defined as `0.1f`. You could of course override this value on your tests. 


```
#define ASSERT_TRUE(expr) do { \
  if (!(expr)) { \
    x_log_error("\t%s:%d: Assertion failed: %s\n", __FILE__, __LINE__, (#expr)); \
    return 1; \
  } \
} while (0)

#define ASSERT_FALSE(expr) ASSERT_TRUE(!(expr))

#define ASSERT_EQ(actual, expected) do { \
  if ((actual) != (expected)) { \
    x_log_error("\t%s:%d: Assertion failed: %s == %s", __FILE__, __LINE__, #actual, #expected); \
    return 1; \
  } \
} while (0)

#define X_TEST_FLOAT_EPSILON 0.1f

#define ASSERT_FLOAT_EQ(actual, expected) do { \
  if (fabs((actual) - (expected)) > X_TEST_FLOAT_EPSILON) { \
    x_log_error("\t%s:%d: Assertion failed: %s == %s", __FILE__, __LINE__, #actual, #expected); \
    return 1; \
  } \
} while (0)

#define ASSERT_NEQ(actual, expected) do { \
  if ((actual) == (expected)) { \
    x_log_error("\t%s:%d: Assertion failed: %s != %s", __FILE__, __LINE__, #actual, #expected); \
    return 1; \
  } \
} while (0)
```

That is really the heart of the “framework.” A test is just a function, and the assertions are just early-return helpers with logging.

The runner itself is a single function: `x_tests_run(STDXTestCase* tests, int32_t num_tests)`. It installs signal handlers for a few common crash signals, loops over the test array, times each test with `XTimer`, prints a colored PASS or FAIL line, counts how many passed, and then prints a final summary. Its return value is also simple: zero when everything passed, non-zero otherwise. 

```
int x_tests_run(STDXTestCase* tests, int32_t num_tests)
{ 

  // We trap signals to be able to report properly in case of crashes
  signal(SIGABRT, s_tests_on_signal); 
  signal(SIGFPE,  s_tests_on_signal);
  signal(SIGILL,  s_tests_on_signal);
  signal(SIGINT,  s_tests_on_signal);
  signal(SIGSEGV, s_tests_on_signal);
  signal(SIGTERM, s_tests_on_signal);

  int32_t passed = 0; 
  double total_time = 0;
  for (int32_t i = 0; i < num_tests; ++i)
  {
    fflush(stdout);

    XTimer timer;
    x_timer_start(&timer);
    int32_t result = tests[i].func();
    XTime elapsed = x_timer_elapsed(&timer);
    const double milliseconds = x_time_milliseconds(elapsed);
    total_time += milliseconds;

    if (result == 0)
    {
      XLOG_WHITE(" [", 0);
      XLOG_GREEN("PASS", 0);
      XLOG_WHITE("]  %d/%d\t %f ms -> %s\n", i+1, num_tests, milliseconds, tests[i].name);
      passed++;
    }
    else
    {
      XLOG_WHITE(" [", 0);
      XLOG_RED("FAIL", 0);
      XLOG_WHITE("]  %d/%d\t %f ms -> %s\n", i+1, num_tests, milliseconds, tests[i].name);
    }
  }

  if (passed == num_tests)
    XLOG_GREEN(" Tests passed: %d / %d  - total time %f ms\n", passed, num_tests, total_time);
  else
    XLOG_RED(" Tests failed: %d / %d - total time %f ms\n", num_tests - passed, num_tests, total_time);

  return passed != num_tests;
}
```

So when I say this library is tiny, I mean it quite literally. Conceptually, it is just this:
 - A test is a function.
 - A test suite is an array of `{ name, function }`.
 - An assertion is an `if` plus a log plus `return 1`.
 - The runner loops, times, prints, counts, and returns success or failure.
 - That is all. And for a lot of C projects, that is exactly enough.


## Writing Tests Feels Like Writing Code

A test is just a function. No hidden lifecycle, no registration magic, no framework-specific structure.

Here is the smallest meaningful example:

```
#define X_IMPL_TEST
#include "stdx_test.h"

static int32_t test_sum(void)
{
  int32_t value = 2 + 2;
  ASSERT_EQ(value, 4);
  return 0;
}

int32_t main(void)
{
  STDXTestCase tests[] =
  {
    X_TEST(test_sum)
  };

  return x_tests_run(tests, sizeof(tests) / sizeof(tests[0]));
}
```

The assertion macros are equally simple. They check a condition, log an error if it fails, and immediately return from the test.

```
static int32_t test_flags(void)
{
  int ready = 1;
  int failed = 0;

  ASSERT_TRUE(ready);
  ASSERT_FALSE(failed);

  return 0;
}
```

For inequality:

```
static int32_t test_ids(void)
{
  int32_t a = 10;
  int32_t b = 20;

  ASSERT_NEQ(a, b);
  return 0;
}
```

And for floating-point values:

```
static int32_t test_float_math(void)
{
  float value = 10.0f / 3.0f;
  ASSERT_FLOAT_EQ(value, 3.3f);
  return 0;
}
```

There is no ceremony here. Just code, conditions, and clear failure paths.

---

## Output

The output of a test run is one of the most important parts of any testing system. It needs to communicate the state of the suite clearly and immediately, without forcing you to dig through logs or interpret unnecessary noise.

The example below comes from the `stdx_arena.h` test suite. It shows what a test run should look like: simple, direct, and informative at a glance.

```
[PASS]  1/13    0.007700 ms -> test_x_arena_create_destroy
[PASS]  2/13    0.013400 ms -> test_x_arena_simple_alloc
[PASS]  3/13    0.023700 ms -> test_x_arena_multi_alloc_same_chunk
[PASS]  4/13    0.012600 ms -> test_x_arena_alloc_triggers_new_chunk
[PASS]  5/13    0.008800 ms -> test_x_arena_alloc_large_block
[PASS]  6/13    0.007100 ms -> test_x_arena_reset_allows_reuse
[PASS]  7/13    0.006400 ms -> test_x_arena_null_and_zero_alloc
[PASS]  8/13    0.007500 ms -> test_x_arena_alignment_respected
[PASS]  9/13    0.031100 ms -> test_x_arena_strdup_copies_and_null_terminates
[PASS]  10/13   0.008300 ms -> test_x_arena_reset_keep_head_frees_extra_chunks
[PASS]  11/13   0.009200 ms -> test_x_arena_trim_keeps_first_n_chunks
[PASS]  12/13   0.008400 ms -> test_x_arena_mark_release_rewinds_and_frees_chunks
[PASS]  13/13   0.011500 ms -> test_x_arena_spike_then_trim_recovers_memory_pressure
Tests passed: 13 / 13  - total time 0.155700 ms
```

Each line tells you exactly what you need to know: which test ran, how long it took, and whether it passed. At the end, a concise summary confirms the overall outcome of the suite and the total execution time. In practice, this is enough to immediately answer the only questions that matter: did everything pass, and how long did it take?

---

## CMake Integration

The build system follows the same philosophy as the library: tests are just executables.
Instead of relying on an external testing framework, I added a small helper function to CMake that builds and tracks test binaries.
Here is the full `create_test` function:

```
function(create_test)
  set(options NORUN)
  set(oneValueArgs TARGET)
  set(multiValueArgs SOURCES LIBRARIES)
  cmake_parse_arguments(ARG "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

  if(NOT ARG_TARGET)
    message(FATAL_ERROR "create_test: TARGET is required")
  endif()
  if(NOT ARG_SOURCES)
    message(FATAL_ERROR "create_test: SOURCES is required")
  endif()

  add_executable(${ARG_TARGET} ${ARG_SOURCES})

  if(DEFINED OUTPUT_NAME_SUFFIX AND NOT OUTPUT_NAME_SUFFIX STREQUAL "")
    set_target_properties(${ARG_TARGET} PROPERTIES OUTPUT_NAME "${ARG_TARGET}${OUTPUT_NAME_SUFFIX}")
  endif()
  if(DEFINED STDX_INCLUDE_DIR AND NOT STDX_INCLUDE_DIR STREQUAL "")
    target_include_directories(${ARG_TARGET} PRIVATE ${STDX_INCLUDE_DIR})
  endif()
  if(ARG_LIBRARIES)
    target_link_libraries(${ARG_TARGET} PRIVATE ${ARG_LIBRARIES})
  endif()

  get_property(_all_targets GLOBAL PROPERTY STDX_ALL_TEST_TARGETS)
  list(APPEND _all_targets ${ARG_TARGET})
  set_property(GLOBAL PROPERTY STDX_ALL_TEST_TARGETS "${_all_targets}")

  get_property(_all_bins GLOBAL PROPERTY STDX_ALL_TEST_BINS)
  list(APPEND _all_bins $<TARGET_FILE:${ARG_TARGET}>)
  set_property(GLOBAL PROPERTY STDX_ALL_TEST_BINS "${_all_bins}")

  if(NOT ARG_NORUN)
    get_property(_run_bins GLOBAL PROPERTY STDX_RUN_TEST_BINS)
    list(APPEND _run_bins $<TARGET_FILE:${ARG_TARGET}>)
    set_property(GLOBAL PROPERTY STDX_RUN_TEST_BINS "${_run_bins}")
  endif()
endfunction()
```

This function creates a normal executable and stores its path so it can be run later. Nothing more.

Then I added a small runner target:

```
function(build_and_run_tests)
  get_property(_all_test_bins GLOBAL PROPERTY STDX_RUN_TEST_BINS)
  set(_all_test_commands "")

  foreach(test_bin ${_all_test_bins})
    list(APPEND _all_test_commands
      COMMAND ${CMAKE_COMMAND} -E echo "Running test: ${test_bin}"
      COMMAND ${test_bin}
    )
  endforeach()

  add_custom_target(all_tests ALL
    ${_all_test_commands}
    COMMENT "Running all unit tests"
  )
endfunction()
```

Each test is registered like this:

```
create_test(TARGET test_math SOURCES tests/test_math.c)
```

And finally:


```
    build_and_run_tests()
```

That is the entire system.

No external test runner. No dependency on a framework. Each test is just a program that returns zero or non-zero. CMake builds them and executes them. The test library handles reporting.

---

## Final Thoughts

What I like about this approach is not just that it is small. It is that it stays aligned with the actual problem.

Most unit tests in C do not need a complex ecosystem. They need a clear way to express expectations, a simple runner, and readable output. This library provides exactly that, and nothing more.

It does not try to abstract C into something else. It embraces it. Tests are functions. Failures are explicit. The build system remains simple. The entire setup can be understood in minutes. That kind of clarity is rare, and worth preserving.

Good software is not the one with the most features. It is the one that solves the problem without pretending the problem is bigger than it really is.
Most of the time, testing in is not a big problem.
