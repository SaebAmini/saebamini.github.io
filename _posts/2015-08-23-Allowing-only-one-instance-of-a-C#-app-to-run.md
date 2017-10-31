---
layout: post
title: Allowing only one instance of a C# application to run
---

Making a singleton application, i.e. preventing users from opening multiple instances of your app, is a common requirement which can be easily implemented using a [_Mutex_](https://msdn.microsoft.com/en-us/library/system.threading.mutex(v=vs.110).aspx).

A Mutex is similar to a C# [lock](https://msdn.microsoft.com/en-us/library/c5kehkcz.aspx), except it can work across multiple processes, i.e. it is a computer-wide lock. Its name comes from the fact that it is useful in coordinating **mut**ually **ex**clusive access to a shared resource.

Let's take a simple Console application as an example:
 
{% highlight C# %}
    class Program
    {
        static void Main()
        {
            // main application entry point
            Console.WriteLine("Hello World!");
            Console.ReadKey();
        }
    }
{% endhighlight %}

Using a Mutex, we can change the above code to allow only a single instance to print `Hello World!` and the subsequent instances to exit immediately:


{% highlight C# %}
static void Main()
{
    // Named Mutexes are available computer-wide. Use a unique name.
    using (var mutex = new Mutex(false, "saebamini.com SingletonApp"))
    {
        // TimeSpan.Zero to test the mutex's signal state and
        // return immediately without blocking
        bool isAnotherInstanceOpen = !mutex.WaitOne(TimeSpan.Zero);
        if (isAnotherInstanceOpen)
        {
            Console.WriteLine("Only one instance of this app is allowed.");
            return;
        }

        // main application entry point
        Console.WriteLine("Hello World!");
        Console.ReadKey();
        mutex.ReleaseMutex();
    }
}
{% endhighlight %}

Note that we've passed `false` for the `initiallyOwned` parameter, because we want to create the mutex in a signaled/ownerless state. The `WaitOne` call later will try to put the mutex in a non-signaled/owned state.

Once an instance of the application is running, the `saebamini.com SingletonApp` Mutex will be owned by that instance, causing further [`WaitOne`](https://msdn.microsoft.com/en-us/library/85bbbxt9(v=vs.110).aspx) calls to evaluate to false until the running instance relinquishes ownership of the mutex by calling `ReleaseMutex`.

Keep in mind that only one thread can own a Mutex object at a time, and just as with the lock statement, it can be released only from the same thread that has obtained it.