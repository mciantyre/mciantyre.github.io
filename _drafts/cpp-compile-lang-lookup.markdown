---
layout: post
title:  "Safe language string selection in C++11"
date: 2016-08-09 20:10:22 -0400
categories: C++
---

I'm working on a legacy C codebase for an embedded system. The devices is sold
in different countries and has to support different languages. Although we can
set a default language at build time, the end-user is able to change the
language by navigating through a few graphical menus. She'll see all the strings
update immediately on the user interface.

As might be expected when dealing with legacy code, my teammates and I ran into
an issue when adding to the codebase. The original designers took the approach
of

- defining all the strings for each language in a C array
- creating an "indexing enumeration" that is used across all arrays to reference
  the same string
- controlling access to the strings through a function that is aware of the
  selected system language

I've provided a simple example below. However, the codebase has hundreds of
strings; the indexing enum is huge. We also deal with at least 8 languages. It's
easy to make a mistake when adding new strings, which we'll get into soon.

```c
// Possible system languages
enum Languages {
    ENGLISH,
    SPANISH,
    NUMBER_OF_LANGUAGES
};

// English strings
char STR_ENG_HELLO[] = {'H', 'e', 'l', 'l', 'o', '\0'};
char STR_ENG_GOODBYE[] = {'G', 'o', 'o', 'd', 'b', 'y', 'e', '\0'};

// Spanish strings
char STR_ESP_HELLO[] = {'H', 'o', 'l', 'a', '\0'};
char STR_ESP_GOODBYE[] = {'A', 'd', 'i', 'o', 's', '\0'};

// The enumeration describes how strings are
// ordered in each array.
enum LanguageString {
    LANG_HELLO,
    LANG_GOODBYE,
    NUMBER_OF_STRINGS
};

// Must maintain the order in each
// language array
static char *ENGLISH_STRINGS[NUMBER_OF_STRINGS] = {
    STR_ENG_HELLO,
    STR_ENG_GOODBYE,
};

static char *SPANISH_STRINGS[NUMBER_OF_STRINGS] = {
    STR_ESP_HELLO,
    STR_ESP_GOODBYE,
};

// Our default language is english
static char **language = ENGLISH_STRINGS;

// Change the system lanuage
void switch_language(enum Languages lang)
{
    switch(lang)
    {
        case ENGLISH: language = ENGLISH_STRINGS; break;
        case SPANISH: language = SPANISH_STRINGS; break;
        // Default is arbitrary
        default:      language = ENGLISH_STRINGS; break;
    }
}

// Get a string depending on the system language
const char* const get_string(enum LanguageString str)
{
    return language[str];
}
```

Accessing strings is mediated through `get_string`. When the user changes the
sytem language, `switch_language` handles the swapping of the language arrays,
and the UI can update with the new language. Subsequent `get_string` invocations
will use the new language array.

The design requires that strings in each language array are in alignment with
the keys in the `LanguageString` enum. If a new string is added in the middle of
the ENGLISH_STRINGS array, but its key is added near the end of the enum, all of
the strings on the UI will be in incorrect positions. We experienced the issue
first-hand when we added new strings and a string key without concern of the
order. It took us more time than it should have to figure out the problem.

As it turns out, the inputs to every invocation of `get_string` are known at
compile time. There isn't a need for `get_string` to be a runtime function; it
could be a compile-time function. More than that, we could use C++
metaprogramming techniques to guarantee correctness of language string
selection. While I cannot propose such a drastic change to an existing C
project, I was still interested in exploring a solution.

## Compile-time string selection

We'll use the full power of C++11/14 for creating a new implementation. We keep
the idea of having a collection of strings from the same language. However,
instead of keeping the indexing enum and string disparate, we pair the two
together in a `LangPair` compile-time struct.

```c++
template <LanguageKey Key, const char* const S>
struct LangPair
{
    static constexpr LanguageKey key = Key;
    static constexpr const char* const value = S;
};
```

(I didn't know a `constexpr const T* const` is a thing until clang guided me to
the solution; we need more `const`...)

The goal, at compile time, is to create a list of `LangPair`s that map a
`LanguageKey` to a string `S` of a particular language. We want something
resembling

```c++
using EnglishStrings
= type_list
<
    LangPair<LanguageKey::LANG_HELLO    , STR_ENG_HELLO>,
    LangPair<LanguageKey::LANG_GOODBYE  , STR_ENG_GOODBYE>
>;

using SpanishStrings
= type_list
<
    LangPair<LanguageKey::LANG_GOODBYE  , STR_ESP_GOODBYE>,
    LangPair<LanguageKey::LANG_HELLO    , STR_ESP_HELLO>
>;
```

Notice that I swapped the order of the strings between the two lists. We'll show
that the design is resiliant to a re-order, and any issues with missing strings
are caught at compile-time.

The `type_list` is simply defined as

```c++
template <typename...>
struct type_list;
```

`type_list` has no definition. It exists to hold a variable number of type
inputs. `LangPair`s are a variety of types, but this `type_list` can also accept
non-homogeneous types. We can pattern-match on a `type_list` at compile time.
Pattern matching closely resembles that of Haskell; you could make a stretch and
say it is also like the spread operator in JavaScript.

We need a way to lookup a key in the `type_list` and get the related value. A
compile-time function `lookup` will have the signature

```c++
template <LanguageKey Key, class TList>
struct lookup;
```

Where the `class TList` will be a `type_list`.