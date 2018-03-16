---
layout: post
title:  "Safer configuration and exception masks in C++"
date:   2018-03-19 9:00:00 -0400
categories: [C++, embedded]
---

In embedded C software, it's typical to see configurations and exceptions represented as bitmasks. The approach allows us to configure ON/OFF setting in a space-efficient manner. We work with the masks using bitwise AND and OR, checking and setting bits as required.

```c
#define	SPI_MODE_CKPOL_HIGH     (1 <<  8)
#define	SPI_MODE_CKPHASE_HALF   (1 <<  9)
#define	SPI_MODE_BODER_MSB      (1 << 10)
#define	SPI_MODE_CSPOL_HIGH     (1 << 11)
#define	SPI_MODE_CSSTAT_HIGH    (1 << 12)
#define	SPI_MODE_CSHOLD_HIGH    (1 << 13)

typedef spi_cfg unsigned int;

int spi_cfg(int fd, int dev, spi_cfg cfg);
```

_Above: real-life SPI configuration bits indended for setting device settings. Device configurations are stored in bits, and the driver checks if these bits are high or low to change the hardware behavior. A typedef loosely associates an unsigned integer with a `spi_cfg` so that we know its a special value._

However, we may run into issues when we forget (or neglect) to type alias our configuration and exception types. I recently ran into some undocumented code resembling

```c
uint32_t validateData(uint8_t* buf, size_t n);
```

What does the return of `validateData` mean? Typically, functions looking like this return 0 on success, or a negative value for an error. But this return is unsigned. Is it a value representing a summary of the input data, like a mean or summation?

In this library, the return is an exception mask. The original implementors track computational errors in a bitmask. The presence of certain exceptions -- or any exceptions -- influences control flow. The team implemented the bitmasks using a C-style enum:

```c
typedef enum err_comp_exceptions {
    COMP_NO_EXCEPTION = 0,
    COMP_DIV_BY_ZERO  = 1,
    COMP_EXCEEDS_MIN  = 2,
    COMP_EXCEEDS_MAX  = 4,
    COMP_NOT_ENOUGH   = 8,
    // Other exceptions...
} err_comp_exceptions_t;
```

I was using `validateData` without issues after realizing the API. I just had to be careful with the return. I shouldn't add it to any other `uint32_t`s, and I shouldn't expect the decimal value to be individually meaningful since it represents many possible values. I should only use the return for checks resembling

```c
uint32_t errs = validateData(buf, sizeof(buf));
if (errs & (COMP_EXCEEDS_MIN | COMP_EXCEEDS_MAX))
{
    // Do stuff when both ranges were exceeded
}
else if (err & COMP_EXCEEDS_MIN)
{
    // Do stuff when min was exceeded
}
else if (err & COMP_EXCEEDS_MAX)
{
    // Do stuff when max was exceeded
}
```

But what if I wasn't careful? I started thinking about the type safety of the API. Nothing _really_ stops me from, for example, adding the exception mask to the index of an array. Even if the API had a C-style `typedef`

```c
typedef validation_err uint32_t;
validation_err validateData(uint8_t* buf, size_t n);
```

I could still perform math with a `validation_err` and other numbers. What would a safer, C++ style exception mask look like? How could it prevent these silly programming errors? And how can we test it?

# Safer `Flag`s and `Mask`s

We can take advantage of a few modern C++ features to implement a safer and testable `Mask<Flag>` type:

- `enum class`, a strongly-typed, namespaced `enum` available since C++11. While a normal `enum` will implicitly convert into its underlying representation (an `int`), an `enum class` does not implicitly convert into a number. We would have to `static_cast` it to turn it into a number. The `static_cast` means "I'm sure I want this to be a number;" it's a decision the programmer, not the compiler, makes. Additionally, we can specify the underlying data type of an `enum class` to control the data size and number of bits for exceptions.
- `constexpr` constructors. A `constexpr` constructor allows a struct / class to exist at compile time, provided all of its underlying members can exist at compile-time. When things can exist at compile-time, we can unit test them at compile-time using `static_asserts`

Let's start by translating the existing `err_comp_exceptions_t` into an `enum class`:

```c++
enum class ValidationExceptions : uint32_t
{
    COMP_NO_EXCEPTION = 0,
    COMP_DIV_BY_ZERO  = 1,
    COMP_EXCEEDS_MIN  = 2,
    COMP_EXCEEDS_MAX  = 4,
    COMP_NOT_ENOUGH   = 8,
    // ...
};
```

The `: uint32_t` syntax looks like class inheritance, but in the context of an `enum class` it defines the size of the underlying data type. In our case, `ValidationExceptions` occupies a `uint32_t`. Everything else is the same.

If we were to use it in the previous context, it would look pretty ugly, since we would have to namespace qualify and typecast the flags...

```c++
uint32_t errs = validateData(buf, sizeof(buf));
// ...
else if (err & static_cast<uint32_t>(ValidationExceptions::COMP_EXCEEDS_MIN))
{
    // Do stuff when min was exceeded
}
// ...
```

We want a way to work directly with the `ValidationExceptions` as if they were flags in a bitmask, so lets represent that in a new type. We want that other type to handle the operations on the mask and the flags. Let's call that type `Mask`, and it looks like this:

```c++
template <typename Flag>
struct Mask
{
    uint32_t mask;
    
    /* 1 */ constexpr Mask()          : mask(0) {}
    /* 2 */ constexpr Mask(Flag flag) : mask(static_cast<uint32_t>(flag)) {}
    
    // ...
};
```

`Mask<Flag>` is a generic bitmask compatible with bitflags of type `Flag`. Our `ValidationExceptions` are a valid type of `Flag` that could plug into `Mask`. We can define a unique type for tracking `ValidationExceptions` as

```c++
using ValidationMask = Mask<ValidationExceptions>;
```

Let's explore how we can work with `Mask`s and `Flag`s as if they were bits and bitmasks. We define two constructors for a `Mask`:

1. The default constructor. It initializes the mask to zero. This is equivalent to the `COMP_NO_EXCEPTION` variant in `ValidationExceptions`. When we default construct a `Mask`, we mean that "there's no exception."
2. The constructor with a flag parameter. We `static_cast` the flag and set the value in `mask`. Based on our implementation, this also becomes the conversion operator from a `Flag` to a `Mask`. This allows a `Flag` to implicitly become a `Mask`, a feature that will be useful as we implement bitwise operators.

When I have a `Mask<Flag>`, I only want to operate on `Flag`s of the same type, or other `Mask<Flag>`s of the same type. I should _not_ be able to AND or OR in any arbitrary numbers, other flags, or other masks. I want to represent these operations on the underlying `mask` member. With this in mind, let's define the compound OR (`|=`) and AND (`&=`) operators. These are like `+=`, but for `|` and `&`.

```c++
template <typename Flag>
struct Mask
{
    // ...

    using This = Mask<Flag>;

    constexpr This& operator|=(This that)
    {
        this->mask |= that.mask;
        return *this;
    }
    
    constexpr This& operator&=(This that)
    {
        this->mask &= that.mask;
        return *this;
    }

    // ...
};
```

We use `using This = Mask<Flag>;` so we don't have to type out `Mask<Flag>` so much. The compound operators work upon the underlying `mask` members on both function arguments. Notice that we're taking `This` by _copy_, not reference. Since we would take any `uint32_t` by copy, and a `Mask<Flag>` is just a `uint32_t` in size, this isn't too big of a deal.

By this point, we can start setting values in exception masks by ORing / ANDing in flags and other masks:

```c++
ValidationMask mask;
if (0 == denom) // prevent divide-by-zero
{
    mask |= ValidationExceptions::COMP_DIV_BY_ZERO;
}

// ...

ValidationMask anotherMask(ValidationExceptions::COMP_NOT_ENOUGH);
mask &= anotherMask;
```

Up above, we OR in a divide-by-zero flag if there was a divide-by-zero condition. Sometime later, we AND with an entirely different mask. We've already achieved the type safety we want, since we cannot OR / AND in arbitrary numbers:

```c++
ValidationMask mask;
if (0 == denom) // prevent divide-by-zero
{
    mask |= (1 << 0); // ERROR: does not compile! 
}

// ...

uint32_t anotherMask = 8;
mask &= anotherMask;    // ERROR: does not compile!
```

In order to check the values in the mask, it would be nice to define the binary operators `|` and `&`. We can express those in terms of the compound operators. (We declare them `friend` so that they're technically defined "outside of the class." We define the returns as `const` following some good [rules of thumb](http://courses.cms.caltech.edu/cs11/material/cpp/donnie/cpp-ops.html).)

```c++
template <typename Flag>
struct Mask
{
    // ...
    
    friend constexpr const This operator|(This lhs, This rhs)
    {
        lhs |= rhs;
        return lhs;
    }
    
    friend constexpr const This operator&(This lhs, This rhs)
    {
        lhs &= rhs;
        return lhs;
    }

    explicit operator bool() const { return mask > 0; }
};
```

We also add the `operator bool()`, allowing our `Mask<Flag>` to convert into a `bool` for use in `if` statements. We convert to `true` if the mask is non-zero, else `false`. With these in place, we can start checking our masks for any exceptions, or for specific flags:

```c++
if (mask & ValidationExceptions::COMP_DIV_BY_ZERO)
{
    // If there was a divide by zero error...
}
else if (mask)
{
    // If there was any error at all...
}
```

# Compile-time unit testing

I love unit testing, even if it's little things like this. Since our `Mask<Flag>` is fully `constexpr`, we can test it at compile time using `static_asserts`. If you go with this approach, I recommend putting these in a relevant source file and not a header so that if they fail, our compile-time errors aren't numerous.

```c++
static_assert(
    (ValidationMask(ValidationExceptions::COMP_NOT_ENOUGH)
     | ValidationExceptions::COMP_DIV_BY_ZERO).mask
    ==
    (static_cast<uint32_t>(ValidationExceptions::COMP_NOT_ENOUGH)
     | static_cast<uint32_t>(ValidationExceptions::COMP_DIV_BY_ZERO)),
    "Failed to OR mask and flag"
);

static_assert(
    (ValidationMask(ValidationExceptions::COMP_NOT_ENOUGH)
     & ValidationExceptions::COMP_DIV_BY_ZERO).mask
    ==
    (static_cast<uint32_t>(ValidationExceptions::COMP_NOT_ENOUGH)
     & static_cast<uint32_t>(ValidationExceptions::COMP_DIV_BY_ZERO)),
    "Failed to AND mask and flag"
);
```

I'll admit: they're not the nicest thing to look at. But we get the guarantees that they work every time we run our code, not just when we run our unit test suite. Notice that we cheat by accessing the intentionally-public `mask` member. Exposing the `mask` member is our escape-hatch if / when we need to interact with existing code. However, seeing the `mask` member out and about should raise some eyebrows, and it's our cue to smell the code a little more...

# Conclusion

We took an API for storing configurations and exceptions in a bitmask and made it type safe. At compile-time, we can catch silly mistakes that wouldn't make sense, like adding an exception mask to another number. We can enforce guarantees about our code, such as a `ValidationMask` that only works with `ValidationExceptions` and not other types of strongly / weakly typed flags. We can refactor our API to be explicit. We can simply change our types, not our implementation, in order to gain the guarantees.

```c
// new signature...
ValidationMask validateData(uint8_t* buf, size_t n);

// old way...
uint32_t mask = 0;
if (0 == denom)
{
    mask |= COMP_DIV_BY_ZERO;
}

// new way...
ValidationMask mask; // change the type (one change)
if (0 == denom)
{
    // qualify the type (another change)
    mask |= ValidationExceptions::COMP_DIV_BY_ZERO;
}
```

# Future work

- `Mask<Flag>` assumes that a `uint32_t` is sufficient to hold all possible `Flag`s. An advanced `Mask<Flag>` implementation could specialize on the underlying `sizeof(Flag)` to pick a template class specialization that uses a smaller / larger underlying type.

# Resources

- [_Why is enum class prefered over plain enum?_](https://stackoverflow.com/questions/18335861/why-is-enum-class-preferred-over-plain-enum)
- [_What are the basic rules and idioms for operator overloading?_](https://stackoverflow.com/questions/4421706/what-are-the-basic-rules-and-idioms-for-operator-overloading/4421719#4421719)
- [_C++ Operator Overloading Guidelines_](http://courses.cms.caltech.edu/cs11/material/cpp/donnie/cpp-ops.html)

# Full code

<script src="https://gist.github.com/mciantyre/cdf0b8e1e74a5ad2cf8d8914938e0ea0.js"></script>