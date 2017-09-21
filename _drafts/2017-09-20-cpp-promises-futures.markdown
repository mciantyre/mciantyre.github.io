---
layout: post
title:  "C++ promises and futures"
date: 2017-09-20 19:27:00 -0400
categories: C++
---

The C++11 standard introduced `std::promise` and `std::future` as a mechanism for concurrent programming. The promise-future pattern is similar to what you find in JavaScript: a "promise" denotes the fulfillment of a computation, and the "future" defines means of accessing the computation's result. The pair can be used for memory-safe communication across threads and for defining asynchronous tasks.

Let's explore `std::promise` and `std::future`, as well as some modern C++ concepts and standard library components, by creating a basic promise-future paring. Warning up front: do not rely on these in your production code! Although they compile and perform, they are not thoroughly tested. `std::promise` and `std::future` will have everything you need, and they are tested, documented, and supported across standard libraries and platforms.

## `our::promise` and `our::future`

Our promises and futures, aptly-named `our::promise` and `our::future`, will support the following simple interface:

- Put a result in the promise with `our::promise::set`
- Check if the future is valid with `our::future::valid`
- Wait on, and receive the value from, the future with `our::future::get`
- Enforce only one owner for a promise and a future.

The API is consistent with the standard library types. We will also design our promise-future pair to have the same behaviors as the `std` equivalents with respect to the interface.

## Where's the value?

We will have to implement all of the memory management for our promise-future pair, which begs the question: what happens when we put a value in the promise? Who owns the data? How does it make its way to `our::future`?

The standard library analogs have a "shared state" to hold the value set on the `promise` and received by the `future`. The standard library "shared state" is tuned for all sorts of use cases. But, for our own exploration, we will define a `our::shared_state` to simply hold an arbitrary value of type `T` and provide some memory-safe behavior:

```c++
namespace our
{
  template <typename T>
  class shared_state
  {
    std::mutex m_mutex;
    std::condition_variable m_cv;
    bool m_ready;
    T m_value;
  public:
    // Create a default state
    shared_state()
      : m_ready(false)
      , m_value{} { /* empty ctor body */ }

    // Set the value
    void set(T&& value)
    {
      std::lock_guard<std::mutex> lock(m_mutex);
      m_value = std::forward<T>(value);
      m_ready = true;
      m_cv.notify_all();
    }

    // Is the shared state ready?
    bool ready()
    {
      std::lock_guard<std::mutex> lock(m_mutex);
      return m_ready;
    }

    // Wait if not ready, then take the value
    T wait_and_take()
    {
      if (m_ready)
        return std::move(m_value);

      std::unique_lock<std::mutex> lock(m_mutex);
      m_cv.wait(lock, [this]() -> bool { return m_ready; });

      return std::move(m_value);
    }
  };
}
```

`our::shared_state` will be an implementation detail for our promise-future pair. It is instantiated in a "not ready" state with a default-constructed value of type `T`. The design assumes that `T`s are default constructible.

`our::shared_state` holds the majority of our memory-safe implementation. The type maintains a `std::mutex` that will be locked when accessing the value and "ready" flags. It also has a `std::condition_variable` that will be waited on if the value is not ready. We can `set` a value in the `shared_state`, check if the value is `ready`, and `wait_and_take` on the shared state. When we `set` the value on the shared state, we lock the mutex, forward the value into the state, and ready the flag. If anyone is waiting on the `conditional_variable`, we wake them up with `notify_all()`.

To perform a `wait_and_take`, we lock the mutex with a `unique_lock` and pass it into `conditional_variable::wait`. `wait` will unblock and re-acquire the lock when (1) the object is notified, and (2) `m_ready` is `true`. Once awoken, we can be sure we have a value, and we move it out of the state as the return of the function.

## The promise

Let's build `our::promise`. We will deviate from the design of the standard library with this one. We will say that a promise is created with a pointer to an `our::shared_state`. It will modify the shared state when setting a value, then it will remove its reference to the shared state:

```c++
namespace our
{
  template <typename T>
  class promise
  {
    std::shared_ptr<our::shared_state<T>> m_shared_state;
  public:
    promise(std::shared_ptr<our::shared_state<T>> shared_state)
      : m_shared_state(std::move(shared_state))
    {
      /* empty ctor body */
    }

    // Our promise will not be copied
    promise(const promise<T>&) = delete;
    promise<T>& operator=(const promise<T>&) = delete;

    // Our promise may be moved
    promise(promise<T>&&) = default;
    promise<T>& operator=(promise<T>&&) = default;

    // Set the value in the promise
    void set(T&& value)
    {
      if (m_shared_state)
      {
        m_shared_state->set(std::forward<T>(value));
        // Remove our reference to the shared state
        m_shared_state = std::shared_ptr<our::shared_state<T>>{};
      }
    }
  };
}
```

`our::promise` accepts a `std::shared_ptr` to one of our shared states. To enforce strict memory ownership, we explicitly `delete` the copy constructor and copy assignment operators. Therefore, we cannot copy our promise and have two promises pointing to the same shared state. We make the move constructor and assignment operators explicit, since it is appropriate to hand-off ownership of a promise to another unit.

When we set the value of the promise, we first check to make sure we have a pointer to our shared state. If we do, we call `set` on the shared state, and forward along the value.

The final action of `set` resets our `shared_ptr` to `nullptr`. Since we null-check our shared state for every `set`, the design guarantees that we can only set the value on the shared-state once. Subsequent invocations to `set` will do nothing.

## The future

Finally, we code our future. Similar to `our::promise`, `our::future` will be constructed with a pointer to a shared state:

```c++
namespace our
{
  template <typename T>
  class future
  {
    std::shared_ptr<our::shared_state<T>> m_shared_state;
  public:
    future(std::shared_ptr<our::shared_state<T>> shared_state)
      : m_shared_state(std::move(shared_state))
    {
      /* empty ctor body */
    }

    // Our future will not be copied
    future(const future<T>&) = delete;
    future<T>& operator=(const future<T>&) = delete;
    
    // Our future may be moved
    future(future<T>&&) = default;
    future<T>& operator=(future<T>&&) = default;
    
    // See if the future is valid
    bool valid() const
    {
      if (!m_shared_state)
        return false; 
      
      return m_shared_state->ready();
    }

    // Get the value
    T get()
    {
      if (!m_shared_state)
        throw std::logic_error("Value already gotten!");

      T value = m_shared_state->wait_and_take();
      // Remove our reference to the shared state
      m_shared_state = std::shared_ptr<our::shared_state<T>>{};
      return std::move(value);
    }
  };
}
```

We can check if the future is `valid`, and we can `get` the value from the future. A `valid` future is one that has a shared state and is also ready. Notice that `get` performs the `wait_and_take` on the shared state, so invocations of `get` will block the calling thread until the future is valid. If a consumer does not want to wait on `get`, she should check `valid` before invoking `get`. Consumers should also call `valid` to avoid the potential exception returned from `get`.

As we did with the promise, we reset our `shared_state` to `nullptr` after we obtain the value. Subsequent invocations of `get` will throw an exception. The behavior is consistent with `std::future`.

## Making promise-future pairs

To ensure a one-to-one pairing of promises to futures, we enforce construction of both with a `make_promise_future` function. The function returns an `std::tuple` that has the promise in the 0th position, and the futures in the 1st position:

```c++
// These convenient compile-time constants index the promise-future
// when used with std::get...
constexpr size_t the_promise = 0;
constexpr size_t the_future = 1;

template <typename T>
std::tuple<our::promise<T>, our::future<T>> make_promise_future()
{
  std::shared_ptr<our::shared_state<T>> shared_state(new our::shared_state<T>);
  return std::make_tuple(our::promise<T>(shared_state), our::future<T>(shared_state));
} 
```

This part of the API is inconsistent with the standard-library components. In the standard library, a future is derived from a promise. It ends up being a cleaner API than what we're implementing. Nevertheless, both designs guarantee that the promise and future share the same state. 

```c++
// Standard library API
std::promise<int> promise;
// A future derived from its promise
std::future<int> future = promise.get_future();
```

After creating a promise-future pair with `make_promise_future()`, we can use the methods of `std::tuple` to take the promise and future, and move them about how we see fit.

```c++
int main()
{

  // Make a promise-future pair that accept / return ints
  auto pair = make_promise_future<int>();

  // Move the promise and future from the tuple...
  // the_promise == 0
  auto promise = std::move(std::get<the_promise>(pair));
  // the_future == 1
  auto future = std::move(std::get<the_future>(pair));

  // Create a thread, and wait on the future
  auto waiting_thread = std::thread([&future]()
  {
    std::cout << "Waiting on future...\n";
    auto value = future.get();
    // Prints 42 after 1 second
    std::cout << "The future returned " << value << std::endl;
  });

  std::cout << "Setting promise in one second...\n";
  std::this_thread::sleep_for(std::chrono::seconds(1));

  promise.set(42);
  std::cout << "Set promise\n";

  waiting_thread.join();

  return 0;
}

```

## Comparisons to `std::promise` and `std::future`

I'm not going to measure performance of `our::promise` and `our::future`, because there's no point! We should always favor `std::promise` and `std::future`, or the equivalents provided by a well-established library, rather than hand-rolling our own. The simple interface and behaviors we did implement are consistent with the standard library components. `std::promise` and `std::future` have a much richer interface that allows for timed promise waits, multiple futures for one promise, and "broken promises." Check out the docs for more information.

`std::promise` and `std::future` are the standard library building blocks for other asynchronous abstractions, including `std::packaged_task` and `std::async`. The latter gives you the feeling of C# / Python co-routines but with all the memory and performance guarantees of C++. Make sure you consult the docs! In particular, the `std::future` returned from `std::async` has special behaviors than a normal `std::future`.

## Remarks

Using C++11 standard library components, we were able to quickly prototype a promise-future pair that works in a simple multi-threaded program. Types like `std::mutex`, the various RAII mutex locks, and `std::thread` allow us to write clean and _portable_ multi-threaded code. `std::conditional_variable` allows us to efficiently wait for state changes across multiple threads. Other types, particularly `std::shared_ptr`, provide strong memory and ownership guarantees. With the design described above, the `our::shared_state` for each promise-future pair would be deallocated as soon as the value passed through the promise and out of the future. We achieved this with one `new` and _zero_ `delete` invocations.

We used types to enforce behaviors with our units. Take a look again at the `our::shared_state` unit. We could technically use the same shared state multiple times to hold different values. However, through the use of types in `our::promise` and `our::future`, we enforced the "use once" behavior when the promise is set and when the future is received. Both actions result in a lost reference to the shared state, which eventually goes out of scope. The design keeps the `our::shared_state` unit flexible for use in other designs, and it makes the promise-future design easy to reason about.

A design may exist that uses `std::atomic_flag` instead of the `std::mutex` and `std::conditional_variable` to achieve a "lock-free" shared state. Take a look at that standard library component to learn about some of the advanced memory synchronization techiques available in modern.