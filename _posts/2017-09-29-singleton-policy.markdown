---
layout: post
title:  "On the singleton pattern in C++: Policy-based design"
date: 2017-09-29 9:00:00 -0400
categories: C++
---

[In a previous post]({% post_url 2017-09-24-cpp-singletons %}), we talked about units that use dependency injection to indirectly access singletons, as opposed to units that directly call on global singletons. In our design, a singleton implements an interface, and units accept a reference to the interface type. The design is easier to test, maintain, and refactor.

In this brief follow-up, we'll explore an alternative approach that applies specifically to C++: templates and "policy-based design." Policy-based design comes from Alexandrescu's _Modern C++ Design_ (2001). It is an approach of designing (families of) classes to one or more "policies," or sets of behaviors. Policies are implemented as templates with static or non-virtual instance methods. In our context, we will design a function, rather than classes, to a "logging policy."

As a quick refresher, we ended up with a design that resembles

```c++
int MyThing::DoSomething(int x, ILogging& logging)
{
  m_value += x;

  if (!logging.Log(LogMsg::Result, m_value))
    return -1

  return m_value;
}
```

Since the method `DoSomething` needs to interact with logging, it accepts a reference to an `ILogging` interface. In production, the `Logging` singleton is passed-into the method. In unit testing, we provide a mocked collaborator in order to assert behavior in a repeatable manner.

The `ILogging` interface is simple:

```c++
enum class LogMsg;

class ILogging
{
public:
    virtual ~ILogging() { /* empty */ }
    virtual bool Log(LogMsg, int) = 0;
};
```

Let's consider a redesign of our `Logging` singleton. Rather than implementing a singleton pattern that ensures the "only one" behavior, let's consider implementing our `Logging` singleton as a class with _only_ static functions and members. It ends up that the `Logging` class with static-only functions looks a lot like the `ILogging` interface! 

```c++
class Logging
{
public:
    static bool Log(LogMsg, int);

private:
    Logging() = default;
    static Logging sm_Instance;
    // ...
};
```

As before with the singleton, there is only one static-only `Logging` class in memory. The naive use of static-only `Logging` would resemble the use of the singleton:

```c++
int MyThing::DoSomething(int x)
{
  m_value += x;

  // :( v-- again, sadness
  if (!Logging::Log(LogMsg::Result, m_value))
    return -1

  return m_value;
}
```

But, if `DoSomething` could accept a "logging policy," it would very closely resemble the solution we made when designing to a run-time interface. We can achieve the analogous design if `DoSomething` accepted a template argument `TLogging`, short for "logging type".

```c++
template <typename TLogging>
int MyThing::DoSomething(int x)
{
  m_value += x;

  // Invoke a static method Log on type "TLogging"
  if (!TLogging::Log(LogMsg::Result, m_value))
    return -1

  return m_value;
}

// Usage...
MyThing thing;
int result = thing.DoSomething<Logging>(42);
```

Take a look at the templated `DoSomething` implementation. It invokes a static method `Log` on `TLogging`. Therefore, any type we provide as a template argument _must implement a static method `Log` with the same signature_. Since our `Logging` module has a static method `Log` with the required signature, this program is valid, and we successfully compile. If we instead passed in an `int` as the template argument to `DoSomething`, the program would fail to compile, since the type `int` does not have a static `Log` method. In essence, the compile-time behavior is exactly the same as when we designed to a run-time interface.

## Remarks

If you like header-only C++ projects, or if virtual methods and run-time interfaces are not your forte, the policy design may work for you. The approach ends up eliminating singletons, opting for static memory allocation. Your application may need more control over when and how singletons are allocated; if so, template-based design as we've described may not satisfy your needs.

While it is possible to unit test functions that accept these templates, it is not as straightforward when testing with a full-fledged mocking framework like Google Mock. Google Mock assumes that your class has _virtual_ methods. In order to mock the policy, we would need to embed a singleton mock into a `MockLogging` class with static-only methods. The static-only methods could forward along to the singleton's mocked methods. This would allow us to expect calls on the mock and control the outputs. However, we end up reintroducing a singleton, but only in our test code instead of our production code.
