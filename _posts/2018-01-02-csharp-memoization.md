---
layout: single
classes: wide
title:  "Memoization in C# - Functional approach"
date:   2018-01-02
categories: csharp memoization functional-programming
---


# Functions and memoization #

While C# is not necessarily a functional language, language constructs such as [Func<>](https://docs.microsoft.com/en-us/dotnet/api/?view=netstandard-2.0&term=Func) and [Action<>](https://docs.microsoft.com/en-us/dotnet/api/?view=netstandard-2.0&term=action) became first-class citizens, making it much easier to write functional style code in C#.

<!--more-->

Certainly, such functionality has been supported previously by means of [delegates](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/delegates/), but introduction of [lambda expressions](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/statements-expressions-operators/lambda-expressions) made it a lot more enjoyable to write.

## Functional primer ##

Let's have some _fun_ with functions first. Here's one which squares a number and returns it:

```cs
Func<int, int> square = n => n * n;
```

So, how it's used? Easy:

Call it directly:

```cs
square(2) // returns 4
```

Or, for example, use it as part of LINQ Select (Map):

```cs
Enumerable.Range(1, 3).Select(x => square(x)); // returns [1,4,9]
```

## So, what the heck is memoization ##

[Wikipedia entry on memoization](https://en.wikipedia.org/wiki/Memoization) says that it is an _optimization technique to speed up programs by storing results of expensive function calls_. Fair enough. But in our previous example there wasn't much speeding up to be done, our function just squares a number, that's it!

Let's make it a bit more difficult. We can emulate computationally (or otherwise) intensive operation by introducing an artificial delay when squaring our number:

```cs
Func<int, int> slowSquare = n =>
{
    Thread.Sleep(100);
    return n * n;
};
```

This function just sleeps for `100ms`, and then returns a result. So how it fares?

Let's write a simple helper method which executes an action and measures how long did execution take:

```cs
static TimeSpan Measure(Action action)
{
    var sw = new Stopwatch();
    sw.Start();

    action();

    sw.Stop();
    return sw.Elapsed;
}
```

Now we can easily measure the performance:

```cs
Measure(() => square(2))        // 00:00:00.0001388
Measure(() => slowSquare(2))    // 00:00:00.1006509

var numbers = Enumerable.Range(1, 10);
Measure(() => numbers.Select(square).ToList())      // 00:00:00.0079551
Measure(() => numbers.Select(slowSquare).ToList())  // 00:00:01.0024892
```

As expected, `slowSquare` is much slower.

So, how do we memoize this function?

### Memoization ##

Turns out, it's surprisingly easy to do that in C# (with some caveats, discussed later).

```cs
public static Func<T, TResult> Memoize<T, TResult>(this Func<T, TResult> f)
{
    var cache = new ConcurrentDictionary<T, TResult>();
    return a => cache.GetOrAdd(a, f);
}
```

What this _extension method_ does that it takes a `Func<,>` and wraps it with a `Func<,>` with same type parameters, but it instead calling it directly it invokes [ConcurrentDictionary.GetOrAdd](https://docs.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentdictionary-2.getoradd?view=netstandard-2.0#System_Collections_Concurrent_ConcurrentDictionary_2_GetOrAdd__0__1_), which provides the exact functionality we need, with minimal overhead and proper threading support for multiple readers or writers.

Cost of this approach is that for each memoized `Func`, we're keeping an additional instance of `ConcurrentDictionary`. This is OK for demo purposes but you may want to consider using shared dictionaries if there are many such functions to be memoized to reduce memory usage overhead.

Let's try it then:

```cs
Measure(() => slowSquare(2));   // 00:00:00.1009680
Measure(() => slowSquare(2));   // 00:00:00.1006473
Measure(() => slowSquare(2));   // 00:00:00.1006373

var memoizedSlow = slowSquare.Memoize();
Measure(() => memoizedSlow(2)); // 00:00:00.1070149
Measure(() => memoizedSlow(2)); // 00:00:00.0005227
Measure(() => memoizedSlow(2)); // 00:00:00.0004159
```

So we can see that after just one call to `memoizedSlow(2)` we're returning value from the cache. Neat!

### Pitfalls ###

#### Lazy initialization ####

While the `Memoize` method given above does work with multiple threads, it does not guarantee a single invocation of a generator given function, it's quite possible when multiple threads call the method that passed function `f` will be called multiple times.

Let's try out current implementation. First, we create a function which just returns a what has been passed to it, but it also counts number of times it has been called.

```cs
int calls = 0;
Func<int, int> identity = n =>
{
    Interlocked.Increment(ref calls);
    return n;
};

var memoized = identity.Memoize();

Parallel.For(0, 1000, i => memoized(0));
Console.WriteLine(calls); // 9
```

Here, we were calling `memoized(0)` a 1000 times (with [Parallel.For](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/how-to-write-a-simple-parallel-for-loop)), but our counter says `9`. Why is that? The `GetOrAdd` on `Dictionary` is called **after** initial function has returned a result, so it just happens that each thread calls it's own function as none are present in the dictionary initially.

This may or may not be a problem depending on your use case, but if it is, we can solve this by wrapping our function in [Lazy<>](https://docs.microsoft.com/en-us/dotnet/api/system.lazy-1?view=netstandard-2.0), which ensures only one initialization is made.

```cs
public static Func<T, TResult> LazyMemoize<T, TResult>(this Func<T, TResult> f)
{
    var cache = new ConcurrentDictionary<T, Lazy<TResult>>();
    return a => cache.GetOrAdd(a, new Lazy<TResult>(() => f(a))).Value;
}
```

Replacing `Memoize` with `LazyMemoize` in our previous example solves the issue with multiple initializations, but it comes with slight performance cost.

```cs
var lazyMemoized = thing.LazyMemoize();

Parallel.For(0, 1000, i => lazyMemoized(0));
Console.WriteLine(calls); // 1 - called just once
```

## IO and Tasks ##

One of the nicer features of C# is it's support for [asynchronous programming](https://docs.microsoft.com/en-us/dotnet/csharp/async) via `async` and `await` (i.e. [promise-based concurrency](https://en.wikipedia.org/wiki/Futures_and_promises) model) keywords. Introduced back in 2012 with C# 5.0, it offers an easy way for everyday developer to write clean and understandable asynchronous code without too much cognitive overhead.

One of the key benefits is avoiding blocking-IO (e.g. file and database access, general networking, etc), which leaves CPU to do other important things while waiting for an interrupt. Good thing is that we can easily memoize those kinds of workloads as well.

A lot of framework code, and various external libraries, expose `Async` methods which return [Task<T>](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task?view=netstandard-2.0) necessary for asynchronous operation, e.g. [HttpClient.GetAsync](https://docs.microsoft.com/en-us/dotnet/api/system.net.http.httpclient.getasync?view=netstandard-2.0).

All we need to do in that case is wrap the method in a function and just memoize it:

```cs
var client = new HttpClient();

Func<string, Task<HttpResponseMessage>> Get = location => client.GetAsync(location);

var memoizedGet = Get.Memoize();
var first = await MeasureAsync(() => memoizedGet("https://www.google.com"));

// returns memoized result
var second = await MeasureAsync(() => memoizedGet("https://www.google.com"));
```

## Expiration ##

Everything so far has been useful, but in most real world scenarios, at least when I/O is involved, you need a expiration for your cache entries.

(Un)fortunately there are many ways this could be done:

- Using a timer to evict stale dictionary entries
- Sorting a dictionary and evicting first/last entries
- Removing entries the moment they are requested

However, `.NET Framework` offers a somewhat suitable replacement in form of `ObjectCache`, and it's implementation `MemoryCache`. [MemoryCache](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.caching.memorycache?view=netframework-4.7.1) can be found under `System.Runtime.Caching`, though if you're running .NET Core, you will need to use [preview NuGet](https://www.nuget.org/packages/System.Runtime.Caching/4.5.0-preview1-25914-04).

---
_Catch you next time!_

## References ##

- [Thread-safe Memoization in C#](https://stackoverflow.com/questions/1254995/thread-safe-memoization)