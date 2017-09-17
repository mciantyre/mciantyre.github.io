---
layout: post
title:  "Safe language string selection in C++11"
date: 2017-09-17 18:29:00 -0400
categories: C++
---

I'm working on a legacy C codebase for an embedded system. The devices is sold
in different countries and has to support different languages. Although we can
set a default language at build time, the end-user is able to change the
language by navigating through a few graphical menus. She'll see all the strings
update immediately on the user interface.

As might be expected when dealing with legacy code, my teammates and I ran into
an issue when adding language strings to the codebase. The original designers
took the approach of

- defining all the strings for each language in a C array
- creating an "indexing enumeration" that is used across all arrays to reference
  the same string
- controlling access to the strings through a function that is aware of the
  selected system language

I've provided a simple example below. However, the codebase has hundreds of
strings; the indexing enum is huge. We also deal with at least 8 languages. It's
easy to make a mistake when adding new strings, as we'll soon see.

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

As it turns out, the outputs from an invocation of `get_string` are known at
compile time. There isn't a need for `get_string` to be a runtime function; it
could be a compile-time function. More than that, we could use C++
metaprogramming techniques to guarantee correctness of language string
selection. While I cannot propose such a drastic change to an existing C
project, I was still interested in exploring a solution.

## Compile-time string selection

We'll use the full power of C++11 to create a new implementation. We keep
the idea of having a collection of strings from the same language. However,
instead of keeping the indexing enum and string separate, we pair the two
together in a `LangPair` compile-time struct.

```c++
template <LanguageKey Key, const char* const S>
struct LangPair
{
    static constexpr LanguageKey key = Key;
    static constexpr const char* const value = S;
};
```

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

We use a `type_list` to create a collection of `LangPair`s. Notice that I
swapped the order of the strings between the two lists. We'll show
that the design is resiliant to a re-order, and any issues with missing strings
are caught at compile-time. (Of course, I can't help you if you pair a "hello"
key with a "goodbye" string!)

The `type_list` is defined as

```c++
template <typename...>
struct type_list;                                   // (1)

template <typename Head, typename ...Tail>
struct type_list<Head, Tail...> : type_list<Head>   // (2)
{
    using tail = type_list<Tail...>;
};

template <typename Head>
struct type_list<Head>                              // (3)
{
    using head = Head;
};
```

`type_list` accepts a variadic number of types (1). One specialization defines a
`type_list` with a head and a variadic tail; that is, one or more tail elements
(2). We use inheritance to say that a `type_list` that matches (2) also has a
head. When there is only one element in the list, the (3) specialization
matches, and it defines the "head" of the list at this point.

Here's a demonstration of accessing the third element of some type list:

```c++
using some_list = type_list<int, bool, float, double, long>;
using third = typename some_list::tail::tail::head;
static_assert(std::is_same<third, float>::value, "It's not a float!?");
```

We can traverse the list by peeling away the list with `tail`, then grab the
head element with `head`.

We need a way to lookup a key in the `type_list` and get the related value. A
compile-time function `lookup` will have the signature

```c++
template
<
    LanguageKey Key,
    class TList,
    bool Found = TList::head::key == Key
>
struct lookup;
```

Where the `class TList` will be a `type_list`. There is no definition yet;
we'll define specializations below.

We declared a `bool Found` to indicate that the key of the head element matches
the key we want. If `Found` is false, we want to keep looking down the tail of
the list:

```c++
template
<
    LanguageKey Key,
    class TList
>
struct lookup<Key, TList, false>
{
    using type =
        typename lookup<
            Key,
            typename TList::tail,
            TList::tail::head::key == Key
        >::type;
};
```

If any `lookup` finds that the head's key matches `Key`, return that type:

```c++
template
<
    LanguageKey Key,
    class TList
>
struct lookup<Key, TList, true>
{
    using type = typename TList::head;
};
```

Finally, we can create a `get_string` function to return language
strings at runtime:

```c++
template <LanguageKey Key>
const char* const get_string(const Language language) noexcept
{
    return (Language::ENGLISH == language) ? lookup<Key, EnglishStrings>::type::value
         : (Language::SPANISH == language) ? lookup<Key, SpanishStrings>::type::value
         // Unreachable, but required. Compilation would fail before this is reached at runtime.
         : "NOT FOUND";
};


static Language m_language = Language::ENGLISH;

int main(int argc, const char * argv[]) {
    
    std::cout << get_string<LanguageKey::LANG_HELLO>(m_language) << std::endl;
    
    // The user "changes languages"...
    m_language = Language::SPANISH;
    
    std::cout << get_string<LanguageKey::LANG_GOODBYE>(m_language) << std::endl;
    return 0;
}
```

`get_string` accepts a `LanguageKey` constant as a type argument, and
a `Language` run-time argument. The run-time argument is the current
system language.

Since string lookup is performed at compile-time, there are instances of
`get_string` for each `LanguageKey` argument provided to the function.
It would be as if we wrote out the following functions manually:

```c++
// When the template argument is LanguageKey::LANG_HELLO...
const char* const get_hello_string(const Language language) noexcept
{
    return (Language::ENGLISH == language) ? STR_ENG_HELLO
         : (Language::SPANISH == language) ? STR_ESP_HELLO
         : "NOT FOUND";
};

// When the template argument is LanguageKey::LANG_GOODBYE...
const char* const get_goodbye_string(const Language language) noexcept
{
    return (Language::ENGLISH == language) ? STR_ENG_GOODBYE
         : (Language::SPANISH == language) ? STR_ESP_GOODBYE
         : "NOT FOUND";
};
```

Most importantly, if you pass in a `LanguageKey` that does not exist in _every
language list_, the build will fail. Suppose we add a key for the word "love":

```c++
using EnglishStrings
= type_list
<
    LangPair<LanguageKey::LANG_HELLO    , STR_ENG_HELLO>,
    LangPair<LanguageKey::LANG_GOODBYE  , STR_ENG_GOODBYE>,
    LangPair<LanguageKey::LANG_LOVE     , STR_ENG_LOVE>
>;

using SpanishStrings
= type_list
<
    LangPair<LanguageKey::LANG_GOODBYE  , STR_ESP_GOODBYE>,
    LangPair<LanguageKey::LANG_HELLO    , STR_ESP_HELLO>
    // TODO add "amor..."
>;

int main(int argc, const char * argv[]) {
    // This doesn't compile!
    std::cout << get_string<LanguageKey::LANG_LOVE>(...) << std::endl;
    return 0;
}
```

Because `SpanishStrings` doesn't have a pairing to `LANG_LOVE`, a developer who
tries `get_string<LanguageKey::LANG_LOVE>(...)` will see a build
error. The specific error will most likely resemble a failed attempt to get a
`tail` type from a `type_list`, since the `lookup` function has exhausted the
type list. Error handling is a big TODO and a future research project for me.
But, as far as "experiential prototypes" go, this was good enough for me (and
better than what's there now).

## What's left?

We took a brittle string indexing system and added compile-time safety with
some added flexibility and maintainability. Improvements could include
better compile-time error handling; there should be some way to say something
to the effect of _"Error: key not found in all language lists"_.

The original implementation had _L_ arrays of _N_ string pointers, where _L_ is
the number of languages and _N_ is the number of strings in a language. The
compile-time approach instead has _N_ functions that can return 1 of _L_
language string variants. The proliferation of functions may or may not be
appropriate given your memory and data constraints.
