---
layout: post
title:  "On the singleton pattern in C++"
date: 2017-09-24 21:23:00 -0400
categories: C++
---

The singleton pattern controls the creation and access of data. It is a popular design pattern that has its merits, particularly if controlling the type's memory allocation is critical. However, as a practitioner of automated software testing and someone who loves functional programming, I cringe when I see a singleton being accessed willy-nilly, just as one would access mutable, global state.

```c++
int MyThing::DoSomething(int x)
{
  m_value += x;

  // :( <-- sadness
  if (!Logging::Instance().Log(LogMsg::Result, m_value))
    return -1

  return m_value;
}
```

## How do I test this?

In this contrived example, `DoSomething` mutates its member variable `m_value` with the input `x`, logs the new member value using the `Logging` singleton, then returns the new member value. If the logging action failed, the return is `-1`. Let's assume that `m_value` is `0` on construction. An automated test, using Google Test, would resemble

```c++
TEST(MyThingTest, DoSomethingLogsOk)
{
  MyThing thing;

  EXPECT_CALL(/* what goes here? */, Log(LogMsg::Result, 42))
    .WillOnce(Return(true));

  ASSERT_EQ(thing.DoSomething(42), 42);
}
```

For reasons described in the requirements, it is critical to test the logging of `m_value` as an effect of `DoSomething`. In a unit test, we cannot call into the actual `Logging` singleton because it introduces non-deterministic behavior. In order to design fast and repeatable tests, we will need to introduce a "mocked" `Logging` singleton. But how can we change the global `Logging` singleton? It's supposed to never change!

## PIMPL singletons and the pointer swap

Assuming your `Logging` singleton follows the PIMPL pattern and has virtual methods, you can replace the contents of the inner `Logging` pointer with a mocked subclass of `Logging`.

```c++
// Static fields...
Logging* Logging::sm_pImpl = nullptr;
std::mutex Logging::sm_mutex;

Logging& Logging::Instance()
{
  if (!sm_pImpl)
  {
    std::lock_guard<std::mutex> lock(sm_mutex);
    if (!sm_pImpl)
    {
      sm_pImpl = new Logging();
    }
  }

  return *sm_pImpl;
}

bool Logging::Log(LogMsg msg, int value)
{
  bool result = false;
  // Write to disk, mutate result...
  return result;
}
```

Here's our rudimentary `Logging` implementation (with elided private copy constructors / assignment operators). You may have seen the double-checked locking pattern similar to the one demonstrated in `Instance()`. As a side note: until C++11, [this C++ code was not portable, and it was not a reliable means of controlling concurrent singleton access](http://www.aristeia.com/Papers/DDJ_Jul_Aug_2004_revised.pdf). This is not the portable, memory-safe implementation either. We won't look at the fix; there's a good read [here](http://preshing.com/20130930/double-checked-locking-is-fixed-in-cpp11/) if you want to learn more.

Normally, the `sm_pImpl` static member pointer is private, which is why you're not going to like this part: we need to make a `MockLoggingAccessor` class a `friend` of `Logging`:

```c++
class Logging
{
  friend class MockLoggingAccessor; // :( <-- more sadness
  static Logging* sm_pImpl;

  // ...
};
```

Now, we can subclass `Logging`, mock the public interface, and implement a `MockLoggingAccessor` to perform a pointer swap:

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
    if (nullptr != Logging::sm_pImpl)
      delete Logging::sm_pImpl

    Logging::sm_pImpl = logging;
  }
};
```

The `MockLoggingAccessor::SetLogging` takes a pointer to a `Logging` type. It replaces the singleton's implementation pointer with the pointer of another `Logging` type. That other `Logging` type will be a `MockLogging` pointer:

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
- the `Instance()` method ensures the pointer is not null. Since we replaced the pointer with our `MockLogging` type via `MockLoggingAccessor::SetLogging`, the pointer is non-null.
- `Instance()` returns our `MockLogging` type.
- We expect a call on `MockLogging::Log`, and we control the return.

## A better way

Was that confusing? Be careful when implementing the accessor for the singleton, and be careful about who owns the `MockLogging` pointer memory. If you delete it, your singleton may later try to delete it which could cause a segfault. Also remember that the approach described above _only_ works if you follow the PIMPL pattern. If your static member is an object, this is a no-go.

Could there be a better, simpler design? Consider a new interface for `MyThing::DoSomething` following the ideas of dependency injection:

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

Better yet, our tests are simplified. We don't need any `MockLoggingAccessor` to mutate a global pointer. We simply pass-in a mock of `ILogging`:

```c++
TEST(MyThingTest, DoSomethingLogsOk)
{
  MockLogging logging;
  // No more MockLoggingAccessor
  MyThing thing;

  EXPECT_CALL(logging, Log(LogMsg::Result, 42))
    .WillOnce(Return(true));

  ASSERT_EQ(thing.DoSomething(42, logging), 42);  // Pass-in logging
}

TEST(MyThingTest, DoSomethingLogsFails)
{
  MockLogging logging;
  MyThing thing;

  EXPECT_CALL(logging, Log(LogMsg::Result, 42))
    .WillOnce(Return(false));

  ASSERT_EQ(thing.DoSomething(42, logging), -1);
}
```

By designing to an interface, rather than grabbing references to global objects, we can simplify our testing. Furthermore, the behavior of our units is explicitly defined through our public interface. Consider the function signatures

```c++
MyThing::DoSomething(int)
MyThing::DoSomething(int, ILogging&)
```

Which function might perform logging? It is obvious that the second function might perform logging, since it accepts a `ILogging` type as an argument. However, if the first function accessed the global `Logging` singleton, it is not immediately obvious that it performs logging. We'll have to look at the implementation or, better yet, the documentation ;)