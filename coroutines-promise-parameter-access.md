<table>
<tr><td>Document number</td><td>P?????R0</td></tr>
<tr><td>Date</td><td>2017-07-08</td></tr>
<tr><td>Reply-to</td><td>Toby Allsopp &lt;toby@mi6.gen.nz&gt;</td></tr>
</table>

# Coroutines: Parameter Access for Promise

## Introduction

The Coroutines PDTS [N4663] allows specialization of `coroutine_traits` based on the types of the coroutine parameters and allows the allocation function used to allocate storage for the coroutine state to be passed the coroutine parameters. However, there are use-cases for coroutines for which it is useful for the promise object to have access to the coroutine parameters. This paper proposes a small change to [N4663] that will allow promise types to optionally receive the coroutine parameters.

[N4663]: https://isocpp.org/files/papers/N4663.pdf

## Motivation and Scope

### Asynchronous Actor Model

Consider a class representing a shared resource that is intended to be accessed from multiple threads. Concurrent access is governed by a mutex contained in the object. Member functions on the object provide asynchronous access to the shared resource; rather than blocking while acquiring the mutex they return a future-like object representing the result of the action.

The following example shows the implenentation of a simple map implemented in this style. The types in the `cppcoro` namespace are available in the [cppcoro] library, but a brief description is copied below:

* `cppcoro::lazy_task<T>`: A lazy_task represents an asynchronous computation that is executed lazily in that the execution of the coroutine does not start until the task is awaited.
* `cppcoro::async_mutex`: Provides a simple mutual exclusion abstraction that allows the caller to 'co_await' the mutex from within a coroutine to suspend the coroutine until the mutex lock is acquired.
* `cppcoro::async_mutex_lock`: An object that holds onto a mutex lock for its lifetime and ensures that the mutex is unlocked when it is destructed.

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

## Design Options

### Use the allocation function

### Promise constructor

## Acknowledgements

* Lewis Baker
* Gor Nishanov
* Eric Fiselier
