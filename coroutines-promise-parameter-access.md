<table>
<tr><td>Document number</td><td>P?????R0</td></tr>
<tr><td>Date</td><td>2017-07-??</td></tr>
<tr><td>Reply-to</td><td>Toby Allsopp &lt;toby@mi6.gen.nz&gt;</td></tr>
</table>

# Coroutines: Parameter Access for Promise Object

## Introduction

The Coroutines PDTS [N4663] allows specialization of `coroutine_traits` based on
the types of the coroutine parameters and allows the allocation function used to
allocate storage for the coroutine state to be passed the coroutine parameters.
However, there are use-cases for coroutines for which it is useful for the
promise object to have access to the coroutine parameters. This paper proposes a
small change to [N4663] that will allow promise types to optionally receive the
coroutine parameters.

[N4663]: https://isocpp.org/files/papers/N4663.pdf

## Motivation and Scope

### Asynchronous Actor Model

Consider a class representing a shared resource that is intended to be accessed
from multiple threads. Concurrent access is governed by a mutex contained in the
object. Member functions on the object provide asynchronous access to the shared
resource; rather than blocking while acquiring the mutex they return a
future-like object representing the result of the action.

The following example shows the implenentation of a simple map implemented in
this style. The types in the `cppcoro` namespace are available in the [cppcoro]
library, but a brief description is copied below:

* `cppcoro::lazy_task<T>`: A lazy_task represents an asynchronous computation
  that is executed lazily in that the execution of the coroutine does not start
  until the task is awaited.
* `cppcoro::async_mutex`: Provides a simple mutual exclusion abstraction that
  allows the caller to 'co_await' the mutex from within a coroutine to suspend
  the coroutine until the mutex lock is acquired.
* `cppcoro::async_mutex_lock`: An object that holds onto a mutex lock for its
  lifetime and ensures that the mutex is unlocked when it is destructed.

[cppcoro]: https://github.com/lewissbaker/cppcoro

```c++
template<typename KEY, typename VALUE>
class concurrent_map
{
public:

  [[nodiscard]]
  cppcoro::lazy_task<> set(KEY key, VALUE value)
  {
    cppcoro::async_mutex_lock lock = co_await m_mutex.lock_async();
		
    m_map[key] = value;
  }

  [[nodiscard]]
  cppcoro::lazy_task<std::optional<VALUE>> try_get(KEY key) const
  {
    cppcoro::async_mutex_lock lock = co_await m_mutex.lock_async();

    auto it = m_map.find(key);
    if (it != m_map.end()) co_return it->second;
    co_return std::nullopt;
  }

private:
  mutable cppcoro::async_mutex m_mutex;
  std::map<KEY, VALUE> m_map;
};
```

Now, it would be very convenient to hoist the code that awaits locking the mutex
into the promise object for the coroutine. This would reduce the chances of
making an error by forgetting to lock the mutex and also reduce the bodies of
the coroutine non-static member functions to their essence.

Here is an excerpt from a promise type that implements the automatic locking and
unlock of the mutex:

```c++
template<typename T>
struct lazy_task_with_lock_promose {
  cppcoro::async_mutex& m_mutex; // how to initialize?
  
  auto initial_suspend() {
    return m_mutex.lock_async();
  }
  
  auto final_suspend() {
    m_mutex.unlock();
    return suspend_never{};
  }
  
  // ...
};
```

The problem is: how does the promise type get a reference to the `async_mutex`
object that is a member of the class containing the coroutine?

## Proposal - use the promise constructor

One way to make the coroutine parameters available to the promise object is to
pass them to the promise constructor. At present, the promise type is required
to be default constructible. We could instead follow the same procedure as when
calling an allocation function and first try to find an overload that accepts
the coroutine parameters as arguments and fall back to the zero-argument case if
there is no such overload.

The above example could then be implemented like so:

```c++
template<typename T>
struct lazy_task_with_lock_promise {
  cppcoro::async_mutex& m_mutex;

  template<typename P1, typename... Pn>
  lazy_task_with_lock_promise(P1&& p1, Pn...&& pn) : m_mutex(p1.m_mutex) {}
  
  // ...
};
```

### Original parameters or copies?

The promise object is constructed after the parameters copies have been made, so
both the copies and the original parameters are viable choices.

## Alternative Solutions

### Use the allocation function

A pointer to the implicit object parameter of a non-static member function
coroutine is passed to the allocation function defined in the promise type if
(a) such an allocation function is declared and (b) allocation of the coroutine
status is required.  If it could be guaranteed that the allocation function
would be called, it would be possible to allocate some extra storage and copy
the implicit object parameter into it such that it could be retrieved from the
promise constructor.

This approach is not tenable because it is not guaranteed that the allocation
function will be called. To quote [N4663] section [dcl.fct.def.coroutine].7
(emphasis added):

> An implementation *may* need to allocate additional storage for a coroutine.

### Additional promise function

An alternative to the constructor-based solution described in the previous
section is to instead pass the coroutine parameters to a non-static member
function of the promise type, if declared, called, for example,
`coroutine_parameters`. This would be less convenient that using the constructor
because the promise type would not be able to use reference members to refer to
coroutine parameters.

## Acknowledgements

* Lewis Baker
* Gor Nishanov
* Eric Fiselier
