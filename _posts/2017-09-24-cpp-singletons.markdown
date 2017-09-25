---
layout: post
title:  "On the singleton pattern in C++"
date: 2017-09-24 21:23:00 -0400
categories: C++
---

The singleton pattern controls the creation and access of data. It is a popular design pattern that I've used and seen used in a variety of software designs. However, I cringe when I see a singleton being accessed willy-nilly, just as one would access mutable, global state.

```c++
int MyThing::DoSomething(int x)
{
  m_value += x;

  // :( v-- sadness
  if (!Logging::Instance().Log(LogMsg::Result, m_value))
    return -1

  return m_value;
}
```

## How do I test this?

In this contrived example, `DoSomething` mutates its member variable `m_value` with the input `x`, logs the new member value using the `Logging` singleton, then returns the new member value. If the logging action failed, the function returns `-1`. Let's assume that `m_value` is `0` on construction. An automated test, using Google Test, would resemble

```c++
TEST(MyThingTest, DoSomethingLogsOk)
{
  MyThing thing;

  EXPECT_CALL(/* what goes here? */, Log(LogMsg::Result, 42))
    .WillOnce(Return(true));

  ASSERT_EQ(thing.DoSomething(42), 42);
}
```

For reasons described in the requirements, it is critical to test the logging of `m_value` as an effect of `DoSomething`. In a unit test, we cannot call into the actual `Logging` singleton because it introduces non-deterministic behavior, and it will flood our log file with test data. We need to introduce a "mocked" `Logging` singleton to design fast and repeatable unit tests. But how can we change the global `Logging` singleton so that we can expect calls? It's supposed to never change!

## Singletons and the pointer swap

Assuming your `Logging` singleton uses a lazily-loaded, static pointer and has virtual methods, you can replace the contents of the inner `Logging` pointer with a mocked subclass of `Logging`.

```c++
// Static private fields...
Logging* Logging::sm_pInstance = nullptr;
std::mutex Logging::sm_mutex;

// Get a reference to the singleton
Logging& Logging::Instance()
{
  if (!sm_pInstance)
  {
    std::lock_guard<std::mutex> lock(sm_mutex);
    if (!sm_pInstance)
    {
      sm_pInstance = new Logging();
    }
  }

  return *sm_pInstance;
}

// Log to disk
bool Logging::Log(LogMsg msg, int value)
{
  bool result = false;
  // Write to disk, mutate result...
  return result;
}
```

Here's our rudimentary `Logging` implementation (with elided private copy constructors / assignment operators). You may have seen the double-checked locking pattern similar to the one demonstrated in `Instance()`. As a side note: until C++11, [this C++ code was not portable, and it was not a reliable means of controlling concurrent singleton access](http://www.aristeia.com/Papers/DDJ_Jul_Aug_2004_revised.pdf). We won't look at the fix; there's a good read [here](http://preshing.com/20130930/double-checked-locking-is-fixed-in-cpp11/) if you want to learn more.

Normally, the `sm_pInstance` static member pointer is private, which is why you're not going to like this part: we need to make a `MockLoggingAccessor` class a `friend` of `Logging`:

```c++
class Logging
{
  friend class MockLoggingAccessor; // :( <-- more sadness
  static Logging* sm_pInstance;

  // ...
};
```

The `MockLoggingAccessor` class will have access to the entire `Logging` singleton in a testing environment. Now, we can subclass `Logging`, mock the public interface using Google Mock, and implement a `MockLoggingAccessor` to perform a pointer swap:

```c++
#include "gmock/gmock.h"

class MockLogging : public Logging
{
public:
  MOCK_METHOD_2(Log, bool(LogMsg, int));
};

class MockLoggingAccessor
{
public:
  static void SetLogging(Logging *logging)
  {
    if (nullptr != Logging::sm_pInstance)
      delete Logging::sm_pInstance

    Logging::sm_pInstance = logging;
  }
};
```

The `MockLoggingAccessor::SetLogging` takes a pointer to a `Logging` type. It replaces the singleton's implementation pointer with the pointer of another `Logging` type. That other `Logging` type will be a `MockLogging` pointer created in unit tests:

```c++
TEST(MyThingTest, DoSomethingLogsOk)
{
  MockLogging* logging = new MockLogging();
  MockLoggingAccessor::SetLogging(logging);

  MyThing thing;

  EXPECT_CALL(*logging, Log(LogMsg::Result, 42))
    .WillOnce(Return(true));

  ASSERT_EQ(thing.DoSomething(42), 42);
}
```

Here's how this is supposed to work:

- `MyThing::DoSomething()` invokes `Logging::Instance()`
- the `Instance()` method checks the static pointer. Since we replaced the pointer with our `MockLogging` type via `MockLoggingAccessor::SetLogging`, the pointer is non-null.
- `Instance()` returns our `MockLogging` type.
- We expect a call on `Log` via the injected `logging` pointer, and we control the return.

Was that confusing? I've maintained a code base in which this was the strategy for testing code that used singletons. The testing scaffolding is tedious, and you have to diligently manage the memory that is swapped into the singleton. It's no fun.

## A better way: DI

Could there be an alternative design? Consider a new interface for `MyThing::DoSomething` that relies on dependency injection:

```c++
int MyThing::DoSomething(int x, ILogging& logging)
{
  m_value += x;

  if (!logging.Log(LogMsg::Result, m_value))
    return -1

  return m_value;
}
```

`DoSomething` now accepts a reference to an `ILogging` type. The `ILogging` interface will be satisfied by the `Logging` singleton:

```c++
class Logging : public ILogging
{
public:
  bool Log(LogMsg, int) override;

  // ...
};
```

We still have all the benefits of the singleton! In production code, we can invoke `DoSomething` with the `Logging` singleton instance, since `Logging` satisfies `ILogging`:

```c++
MyThing thing;
int result = thing.DoSomething(2, Logging::Instance());
```

In test code, we don't need `MockLoggingAccessor` to mutate a global pointer. We simply pass-in a mock of `ILogging`:

```c++
TEST(MyThingTest, DoSomethingLogsOk)
{
  MockLogging logging;  // MockLogging subclasses ILogging
  MyThing thing;

  EXPECT_CALL(logging, Log(LogMsg::Result, 42))
    .WillOnce(Return(true));                       

  ASSERT_EQ(thing.DoSomething(42, logging), 42); // Pass-in logging
}

TEST(MyThingTest, DoSomethingLogsFails)
{
  MockLogging logging;
  MyThing thing;

  EXPECT_CALL(logging, Log(LogMsg::Result, 42))
    .WillOnce(Return(false));                 

  ASSERT_EQ(thing.DoSomething(42, logging), -1); // Pass-in logging
}
```

By designing to an interface, rather than grabbing references to global objects, we can simplify our testing. Furthermore, the behavior of our units is explicitly defined in our public interface. Consider the function signatures

```c++
MyThing::DoSomething(int)             // 1
MyThing::DoSomething(int, ILogging&)  // 2
```

Which function might perform logging? It is obvious that the second function might perform logging, since it accepts a `ILogging` type as an argument. However, if the first function accessed the global `Logging` singleton, it is not immediately obvious that it performs logging. We will have to look at the implementation or, better yet, the documentation ;)

Singletons themselves are not bad. Controlling memory allocation and access has objective benefits. However, consumers of your singleton _don't need to know that it is a singleton._ By designing to an interface, we specify our unit behaviors in the public interface, and we create units that are easy to test, refactor, and maintain.
