---
layout: post
title: Allowing only one instance of a C# application to run
---

Making a singleton application, i.e. preventing users from opening multiple instances of your app, is a common requirement, and it can be implemented easily using a [_Mutex_](https://msdn.microsoft.com/en-us/library/system.threading.mutex(v=vs.110).aspx).

A Mutex is much like a C# [lock](https://msdn.microsoft.com/en-us/library/c5kehkcz.aspx), except it can work across multiple processes, i.e. it is a computer-wide lock. Its name comes from the fact that it is useful in coordinating mutually exclusive access to a shared resource.

Suppose we have a WinForms application, the UI is typically started from a Program.cs file like this:
 
{% highlight C# %}
static void Main()
{
    Application.EnableVisualStyles();
    Application.SetCompatibleTextRenderingDefault(false);
    Application.Run(new Form1());
}
{% endhighlight %}

Using a Mutex, we can change the above code to allow only a single instance:
 
{% highlight C# %}
static void Main()
{
    // Named Mutexes are available computer-wide. Use a unique name.
    using (var mutex = new Mutex(false, "saebamini.com SingletonApp"))
    {
        bool isAnotherInstanceOpen = !mutex.WaitOne(TimeSpan.Zero, false);
        if (isAnotherInstanceOpen)
        {
            Console.WriteLine("Only one instance of this app is allowed.");
        }
        else
        {
            RunProgram();
        }
    }
}

static void RunProgram()
{
    Application.EnableVisualStyles();
    Application.SetCompatibleTextRenderingDefault(false);
    Application.Run(new Form1());
}
{% endhighlight %}

Now, once an instance of our application is running, the `saebamini.com SingletonApp` Mutex will be owned (or in a nonsignaled state) by that instance, causing further [`mutex.WaitOne`](https://msdn.microsoft.com/en-us/library/85bbbxt9(v=vs.110).aspx) calls to evaluate to false while it is running or until it calls ReleaseMutex on its Mutex object.

Keep in mind that only one thread at a time can own a Mutex object, and just as with the lock statement, it can be released only from the same thread that obtained it.