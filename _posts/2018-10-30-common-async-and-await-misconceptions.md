---
layout: post
title: Common C# async and await misconceptions
redirect_from:
  - /some-common-.net-asynchronous-programming-misconceptions/
---

.NET Programmers have traditionally shied away from writing asynchronous code, and mostly for good reason. Writing asynchronous code used to be [arduous work](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/asynchronous-programming-model-apm) and the result was [difficult to reason about, debug and maintain](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/event-based-asynchronous-pattern-eap). That became exacerbated when you threw concurrency into the mix – parallel or asynchronous – as that's harder to consciously follow for our brains which are optimised/trained for non-concurrent and sequential logic.

The compiler magic of `async/await` since C# 5.0 and the [Task-based Asynchronous Pattern](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap) has immensely improved that experience, mainly by abstracting asynchronous code to read like synchronous code. In fact it's been such a big hit that it's spread and [been adopted by many other popular languages such as JavaScript, Dart, Python, Scala and C++](https://en.wikipedia.org/wiki/Futures_and_promises#History).

This has resulted in a trend of more and more asynchronous code popping up in our code-bases. That's mostly a good thing, because leveraging asynchrony _in the right place_ can lead to significant performance and scalability improvements. However, with all great magic, comes great misconception. I'll go over some of the most common ones below which I encounter often<!--more-->.

## `async/await` means multi-threading

> "The method isn't `async`, it's single-threaded." – too many devs

I continue to hear and see many varieties of the above statement and it's the most common misconception in my experience. Overloading of terms like _asynchronous_, _concurrent_ and _parallel_ and using them interchangeably in the industry can take some of the blame, and maybe some of it falls on Tasks being included in the `System.Threading` namespace.

`async` doesn't magically make your code asynchronous. It don't spin up worker threads behind your back either. In fact, it doesn't really do anything other than enable the use of the `await` keyword in a method. It was only introduced [to not break existing codebases that had used `await` as an identifier and to make `await` usage heuristics simpler](https://blogs.msdn.microsoft.com/ericlippert/2010/11/11/asynchrony-in-c-5-part-six-whither-async/). That's it!

`await` is a bit more complicated and is quite similar to how the `yield` keyword works, in that it yields flow of control back to the caller and creates a state-machine by causing the compiler to register the rest of the async method as a continuation. That continuation is run whenever the awaited task completes. The compiler transforms the below:

{% highlight C# %}
var result = await task;
statements
{% endhighlight %}

Into essentially:

{% highlight C# %}
var awaiter = task.GetAwaiter();
awaiter.OnCompleted (() =>
{
	var result = awaiter.GetResult();
	statements
});
{% endhighlight %}

That's the gist. The compiler emits [more complex code](https://weblogs.asp.net/dixin/understanding-c-sharp-async-await-1-compilation), however, none of that involves spinning worker threads. It also doesn't magically make your task's implementation asynchronous either. In fact, if the implementation is synchronous it all runs synchronously, but [_slooower_](#all-code-should-be-async-by-default-for-better-performance).

## You can't have concurrency with purely asynchronous code

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Don&#39;t wanna be that guy, but you mean concurrent. Async = non-blocking.</p>&mdash; Diogo Castro (@dfacastro) <a href="https://twitter.com/dfacastro/status/808345214725275649?ref_src=twsrc%5Etfw">December 12, 2016</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

So I just said async doesn't require multiple threads. That means unless you mix in multi-threading you can't have concurrency right?

_Wrong._

Truly asynchronous operations can be _concurrently_ performed by one thread. Heresy! how can one thread perform multiple operations at the same time?

That's possible because IO operations leverage [_IOCP_](https://docs.microsoft.com/en-us/windows/desktop/fileio/i-o-completion-ports) ports at the device driver level. The point of IO completion ports is you can bind thousands of IO handles to a single one, effectively using a single thread to drive thousands of asynchronous operations concurrently.

Take this sample code that makes 5 HTTP calls to a slow API one by one:

{% highlight C# %}
async Task DoSequentialAsync()
{
	const string SlowApiUrl = new Uri("http://slow-api-that-responds-after-2-seconds");
	for (int i = 0; i < 5; i++)
	{
		var client = new WebClient();
		await client.DownloadStringTaskAsync(SlowApiUrl);
	}
}
{% endhighlight %}

For me it takes about a little more than 10 seconds to complete, while using a single thread. No surprise here.

Now what do you think happens if we change the code to instead of awaiting each call one by one, fire off all of them and await the combination of all those tasks?

{% highlight C# %}
async Task DoConcurrentAsync()
{
	const string SlowApiUrl = new Uri("http://slow-api-that-responds-after-2-seconds");
	var tasks = new List<Task>();
	for (int i = 0; i < 5; i++)
	{
		var client = new WebClient();
		var no = i;
		tasks.Add(client.DownloadStringTaskAsync(SlowApiUrl));
	}

	await Task.WhenAll(tasks);
}
{% endhighlight %}

The moment of truth! the whole thing takes a little more than 2 seconds this time, and all with just a single thread. [Here's a more complete example](https://gist.github.com/SaebAmini/05a2370ee8ad7248e2e037ee3b6ffbae) that keeps track of used threads and the exact time. Go ahead and run it for yourself, may you become a believer.

How that's possible is that as soon as each call is fired and registered on a completion port, while it is "in flight", we get the thread back and use it to fire the next one. That's also the same reason why doing something like the above with tasks that use an EF context to query the database asynchronously will cause EF to blow up with a `DbConcurrencyException` saying concurrent queries on the same context aren't supported.

## All code should be `async` by default for better performance

![](/images/posts/async-misconceptions/async_all_the_things.jpg)

We saw that `async/await` generates a state-machine and significant extra code which involves a new type with extra heap allocations, exceptions and try-catches and context marshalling between different execution contexts. As you can imagine, all of that comes with some overhead which inevitably makes asynchronous code run slower than a synchronous counterpart.

If it's slower, why do we use asynchronous code?

In rich client applications which have a UI thread, or single-threaded applications, it's for _responsiveness_. Because you won't block the main thread and freeze the app while waiting for an asynchronous operation.

It's a different story in ASP.NET web applications and APIs. ASP.NET request processing is multi-threaded with a thread per request model. Releasing the thread when awaiting an asynchronous request in an API call doesn't create a more responsive experience for the user because it still has to complete before the user gets the response. The benefit here is _scalability_ and _throughput_ – getting more bang (request handling) for your buck (hardware resources).

Threads are maintained in a thread pool and map to limited system resources. Depending on many different factors, the underlying scheduler might decide to spin more threads and add to this pool which involves some microseconds of overhead per thread. That may seem insignificant, but for an ASP.NET API serving a lot of users, it can quickly add up to significant numbers. By releasing the thread when it's idling for an IO-bound work to complete, such as calling another API, it can be used to serve another request. It also protects against usage bursts since the scheduler doesn't suddenly find itself starved of threads to serve new requests.

This means your API can make better use of available system resources and deal with a lot more requests before it falls over. However, you only get these benefits when you leverage _truly_ asynchronous code on _inherently_ asynchronous IO bound operations that have _significant latency_ like reading a file from a slow drive or making a network call.

If async code is used for performing synchronous operations, or even asynchronous operations that are guaranteed to complete in milliseconds, the overhead outweighs the gains and results in worse performance. [In one benchmark this detriment was measured to be about 300% in total execution time, 600% in memory and 25% in CPU usage](http://www.ducons.com/blog/tests-and-thoughts-on-asynchronous-io-vs-multithreading). So give it a second think before making that next method `async` by default. Horses for courses.

## You need to use `async/await` in the whole call stack

Have you seen code like this before?

{% highlight C# %}
// FooController.cs

public class FooController
{
	public async Task<Foo> GetFoo()
	{
		return await _fooService.GetFooAsync()
	}
}


// FooService.cs

public class FooService
{
	public async Task<Foo> GetFooAsync()
	{
		return await _fooContext.GetAsync()
	}
}

{% endhighlight %}

You should have asynchronous code in the whole call stack to not hold up threads, but you don't have to actually use the `async/await` keywords unless you need to.

`FooController.GetFoo` and `FooService.GetFooAsync` just return another task without doing anything else. These "proxy" methods don't need to use `async/await` at all and can be simplified to:

{% highlight C# %}
// FooController.cs

public class FooController
{
	public Task<Foo> GetFoo()
	{
		return _fooService.GetFooAsync()
	}
}


// FooService.cs

public class FooService
{
	public Task<Foo> GetFooAsync()
	{
		return _fooContext.GetAsync()
	}
}
{% endhighlight %}

It is still asynchronous code, but it's more efficient as it avoids [creating an unnecessary state-machine](#all-code-should-be-async-by-default-for-better-performance). It is also less code and reads better, so it's simply cleaner code.

Note that there are some gotchas involved with omitting the async/await keywords in more complex methods. For example, returning a Task inside a `using` block without awaiting it, exits the block and results in immediate disposal which might not be expected. Also, exception behaviour is slightly different when a synchronous part of the method throws an exception, and the calling code happens to await the method after invoking it, e.g. firing a bunch of tasks and then awaiting them all with `Task.WhenAll`. That exception will be thrown synchronously rather than when the task is awaited. To avoid these issues and their cognitive overhead, I recommend only omitting the async/await keywords in simple proxy methods as my examples above.

&nbsp;

Hopefully this helps debunk some of the most common misconceptions about the magic of `async/await` in C# and .NET.
