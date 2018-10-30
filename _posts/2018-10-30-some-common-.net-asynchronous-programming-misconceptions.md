---
layout: post
title: Misconceptions Some Programming Common Asynchronous .NET 
---

Now that I have my cheesy word-play on the title, to demonstrate [one inherent trait of asynchronous programming](#you-cant-have-concurrency-with-purely-asynchronous-code), out of the way, let's talk about asynchronous programming in C# and .NET and some of the common misconceptions around it.

.NET Programmers have traditionally shied away from writing asynchronous code, mostly for good reason. Writing asynchronous code used to be [arduous work](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/event-based-asynchronous-pattern-eap) and the result was [difficult to reason about, debug and maintain](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/asynchronous-programming-model-apm). That became exacerbated when you threw concurrency into the mix – parallel or asynchronous – as that's harder to consciously follow for our brains which are, or at least trained to be optimised for, non-concurrent and sequential logic.

The compiler magic of `async/await` since C# 5.0 and the [Task-based Asynchronous Pattern](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap) has immensely improved that experience, mainly by abstracting asynchronous code to read like synchronous code. In fact it's been so successful that it's spread and [been adopted by many other popular languages such as JavaScript, Dart, Python, Scala and C++](https://en.wikipedia.org/wiki/Futures_and_promises#History).

This has resulted in a trend of more and more asynchronous code popping up in our code-bases. That's mostly a good thing, because leveraging asynchrony _in the right place_ can lead to significant performance and scalability improvements. However, like any other magic, not being aware of what goes on under the hood can lead to myths, misuse or gotchas that end up biting you. I'll go over some of them below which I encounter often in my consulting work.

## `async/await` requires multi-threading

> "Async is not worth dealing with the multi-threading issues!"

This has likely been the most common misconception in my experience. I've heard many varieties of the above statement over the years. Overloading of terms like asynchronous, concurrent and parallel and incorrectly using them interchangeably can take some of the blame. Although, the situation has improved a lot recently, thanks in large part to Stephen Cleary and his continued evangelism of [`async/await` best practices in .NET](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html) and [how the internals work](https://blog.stephencleary.com/2012/02/async-and-await.html) which I highly recommend reading.

`async` doesn't magically make your code asynchronous. It don't spin up worker threads behind your back either. In fact, it doesn't really do anything, other than enable the use of the `await` keyword in a method. It was only introduced [to not break existing codebases that had used `await` as an identifier and to make `await` usage heuristics simpler](https://blogs.msdn.microsoft.com/ericlippert/2010/11/11/asynchrony-in-c-5-part-six-whither-async/). That. Is. _It_.

`await` is a bit more complicated and is quite similar to how the `yield` keyword works, in that it yields flow of control back to the caller and creates a state-machine by causing the compiler to register the rest of the async method as a continuation. That continuation is run whenever the awaited task completes.

The compiler transforms this:

{% highlight C# %}
var result = await task;
statements
{% endhighlight %}

Into something that's functionally similar to:

{% highlight C# %}
var awaiter = task.GetAwaiter();
awaiter.OnCompleted (() =>
{
	var result = awaiter.GetResult();
	statements
});
{% endhighlight %}

_That's the gist. But the compiler actually emits [slightly more complex code](https://weblogs.asp.net/dixin/understanding-c-sharp-async-await-1-compilation) which also includes short-circuiting the continuation when a task synchronously completes._

None of that involves spinning worker threads. It also doesn't magically make your task's implementation asynchronous. In fact, if the implementation is synchronous it all runs synchronously, but _slower_, which brings us to the next misconception.

## `async` all the things because it is faster

![](/images/posts/async-misconceptions/async_all_the_things.jpg)

As we saw in the previous section, `async/await` generates a state-machine and significant extra code which involves extra heap allocations and context switches between different execution contexts. As you can imagine, all of that comes with some overhead which inevitably makes asynchronous code run slower than a synchronous counterpart.

That's not to dismiss the performance benefits that can come with correctly used asynchronous code, which can be significant. However those benefits aren't about execution time.

In rich client applications which have a UI thread, or single-threaded applications, it's about _responsiveness_. Because you don't block the main thread while waiting for an inherently asynchronous operation and freeze the app.

It's a different story in ASP.NET web applications and APIs. ASP.NET request processing is inherently multi-threaded with a thread per request model. Releasing the thread when awaiting an asynchronous request in an API call doesn't _directly_ create a more responsive experience for the user because it still has to complete before the user gets the response. The benefit here is about _scalability_ and _throughput_.

Threads are maintained in a thread pool and map to limited system resources. Depending on many different factors, the underlying scheduler might decide to spin more threads and add to this pool which involves some microseconds of overhead per thread. That may seem insignificant, but for an ASP.NET API serving a lot of users, it can quickly add up to significant numbers. By releasing the thread when it's idling for an IO-bound work to complete, it can be used to serve another request. It also protects against usage bursts since the scheduler doesn't suddenly find itself starved of threads to serve new requests when it will have to continuously create new ones.

This means your API can make the most use of available system resources and deal with a lot more requests before it falls over. However, you only get these benefits when you leverage _truly_ asynchronous code on _inherently_ asynchronous IO bound operations that have a _significant latency_ like reading a file or making a network call.

If asynchronous code is used for processing synchronous operations, or even asynchronous operations that are guaranteed to complete in less than a few milliseconds, the overhead will outweigh the gains and result in a worse performing API. [In one benchmark this detriment was measured to be about 300% in total execution time, 600% in memory and 25% in CPU usage](http://www.ducons.com/blog/tests-and-thoughts-on-asynchronous-io-vs-multithreading). So give it a second think before making that next method `async` by default. Horses for courses.

## You can't have concurrency with purely asynchronous code

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Don&#39;t wanna be that guy, but you mean concurrent. Async = non-blocking.</p>&mdash; Diogo Castro (@dfacastro) <a href="https://twitter.com/dfacastro/status/808345214725275649?ref_src=twsrc%5Etfw">December 12, 2016</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Since asynchronous code does not involve worker threads then that means unless you mix in multi-threading you can't have concurrency right?

_Wrong._

Truly asynchronous operations can be _concurrently_ processed by one thread. Heresy you say? How can one thread process multiple operations at the same time!

That's possible because IO operations leverage [_IOCP_](https://docs.microsoft.com/en-us/windows/desktop/fileio/i-o-completion-ports) ports at the device driver level. The point of IO completion ports is you can bind thousands of IO handles to a single one, effectively using a single thread to drive thousands of asynchronous operations concurrently.

Take this sample code that makes 5 HTTP calls one by one:

{% highlight C# %}
async Task DoAsyncSequentially()
{
	var client = new HttpClient();
	var sw = Stopwatch.StartNew();
	for (int i = 0; i < 5; i++)
	{
		await client.GetAsync("https://google.com");
	}
	sw.Stop();
	Console.WriteLine(sw.ElapsedMilliseconds);
}
{% endhighlight %}

For me it prints about 1500 on average. What do you think happens if we change the code to instead of awaiting each call one by one, fire off all of them and await the combination of all those tasks?

{% highlight C# %}
async Task DoAsyncConcurrently()
{
	var client = new HttpClient();
	var sw = Stopwatch.StartNew();
	await Task.WhenAll(new[]
	{
		client.GetAsync("https://google.com"),
		client.GetAsync("https://google.com"),
		client.GetAsync("https://google.com"),
		client.GetAsync("https://google.com"),
		client.GetAsync("https://google.com"),
	});
	sw.Stop();
	Console.WriteLine(sw.ElapsedMilliseconds);
}
{% endhighlight %}

The moment of truth!

For me it prints on average about 300. All with one worker thread. Go ahead and run it for yourself.

That's because as soon as each call is fired and registered on a completion port, while it is "in flight", the next one fires. That's also the same reason why doing something like the above with tasks that use an EF context to query the database will cause EF to blow up with a `DbConcurrencyException` saying concurrent queries on the same context aren't supported.

## You need to use `async/await` in the whole call stack

I'm pretty sure something like this looks familiar to a lot of you:

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


// FooContext

public class FooContext
{
	public async Task<Foo> GetAsync()
	{
		statements
		var foo = await GetAsync()
		DoSomethingWithFoo(foo);
		return foo;
	}
}
{% endhighlight %}

While it's a good idea to have asynchronous code in the whole call stack so you don't have blocking code that defeats the purpose of going async, you don't have to actually use the `async/await` keywords unless you need to.

`FooController.GetFoo` and `FooService.GetFooAsync`, which just return another task without performing any additional logic on the result of that task, don't need to use `async/await` at all. They can be simplified to:

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

This isn't just for aesthetics. The simplified version is actually more efficient. Because it avoids [creating an unnecessary state-machine and the extra heap allocations](#async-all-the-things-because-it-is-faster).

&nbsp;

Hopefully this helps debunk some of the most common misconceptions about the magic of `async/await` in C# and .NET.
