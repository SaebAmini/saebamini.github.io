---
layout: post
title: Profiling your ASP.NET API using the Visual Studio performance profiler
---

Recently I needed to profile a client's ASP.NET Core API because it was being very flaky. It would intermittently become very slow and sometimes completely unresponsive, even under normal usage.

So I decided to revisit the tools I normally use to see if there's anything better out there. My aim is to capture and share my findings in this post.

It's worthy to note that while in my case I was dealing with an ASP.NET Core API, a lot of this applies to other .NET applications as well.


## Visual Studio 2015+ performance profiler

Before VS 2015, unless you were profiling for memory issues in which case you could use the [CLR Memory Profiler](https://www.microsoft.com/en-us/download/details.aspx?id=16273), you had to use external tools for profiling. [EQATEC profiler][1]/[JustTrace][1], [ANTS][2] and [dotTrace][3] have all been popular options. And while some of them are still going strong, EQATEC/JustTrace has been retired since its functionalities has been included in VS2015.

[1]: http://www.eqatec.com/Profiler/
[2]: http://www.red-gate.com/products/ants_performance_profiler/
[3]: http://www.jetbrains.com/profiler/

So let's delve into using the tools provided since VS 2015 and see how it compares.