---
layout: post
title: The Dangerous EF Core Feature&#58; Automatic Client Evaluation
---

#### _Update: starting from EF Core 3.0-preview 4, [this damaging default behavior has been greatly limited, although not completely turned off](https://docs.microsoft.com/en-us/ef/core/what-is-new/ef-core-3.0/breaking-changes#linq-queries-are-no-longer-evaluated-on-the-client)._

Recently when going through our shiny new ASP.NET Core application's logs, I spotted a few entries like this:

> The LINQ expression 'foo' could not be translated and will be evaluated locally.

Odd.

I dug around in the code and found the responsible queries. Some of them were quite complex with many joins and groupings, while some of the other ones were quite simple<!--more-->, like `someStringField.Contains("bar", StringComparison.OrdinalIgnoreCase)`.

You may have spotted the problem right away. `StringComparison.OrdinalIgnoreCase` is a .NET concept. It doesn't translate to SQL and EF Core can't be blamed for that. In fact, if you run the same query in the classic Entity Framework, you'll get a `NotSupportedException` telling you it can't convert your perdicate to a SQL expression and that's a _good_ thing, because it prompts you to review your query and if you _really_ want to have a predicate in your query that only applies in the CLR world, _you_ can decide if it makes sense in your case to do a `ToList()` or similar at some point in your `IQueryable` to pull down the results of your query from the database into memory, or you may decide that you don't need that `StringComparison.OrdinalIgnoreCase` after all, because your database collation is case-insensitive anyway.

The point is that, by default _you_ are in control and can make explicit decisions based on your circumstances.

That's unfortunately not the case in Entity Framework Core because of its concept of [mixed client/server evaluation](https://docs.microsoft.com/en-us/ef/core/querying/client-eval). What mixed evaluation effectively does is, if you have anything in an `IQueryable` LINQ query that can't be translated to SQL, it tries to magically and silently make it work for you, by taking the untranslatable bits out and running them locally... and it's enabled by _default_! what could go wrong?

That's an extremely dangerous behavior change compared to the good old Entity Framework. Consider this entity:

{% highlight C# %}

public class Person
{
	public string FirstName { get; set; }
	public string LastName { get; set; }
	public List<Address> Addresses { get; set; }
	public List<Order> Orders { get; set; }
}

{% endhighlight %}

And this query:

{% highlight C# %}

var results = dbContext.Persons
	.Include(p => p.Addresses)
	.Include(p => p.Orders)
	.Where(p => p.LastName.Equals("Amini", StringComparison.OrdinalIgnoreCase))
	.ToList();

{% endhighlight %}

EF Core can't translate `p.LastName.Equals("Amini", StringComparison.OrdinalIgnoreCase)` into a query that can be run on the database, so it pulls down the _whole_ Persons table, as well as the _whole_ Orders and Addresses tables from the database into the memory and _then_ runs the `.Where(p => p.LastName.Equals("Amini", StringComparison.OrdinalIgnoreCase))` filter on the results <span style="font-size: 3em">🤦🏻‍♂️</span>

The fact that this behavior is enabled by default is _mind blowing_! It's not hard to imagine the performance repercussions of that on any real-size application with significant usage. It can easily bring down applications to their knees. Frameworks should make it difficult to make mistakes, especially ones with potentially devastating consequences like this.

You might be thinking that it's the developer's fault for including something like `StringComparison.OrdinalIgnoreCase` in the IQueryable prediate, but having untranslatable things like that in your query isn't the only culprit that results in client evaluation. If you [have](https://github.com/aspnet/EntityFrameworkCore/issues/6245) too many [joins](https://stackoverflow.com/questions/45237492/ef-core-could-not-be-translated-and-will-be-evaluated-locally) or [groupings](https://github.com/aspnet/EntityFrameworkCore/issues/12255), the query could become too complex for EF Core and make it fall back to local evaluation.

So if you're using EF Core, you want to keep an eye on your logs to spot client evaluations. If you don't want that additional cognitive overhead, you can disable it altogether and make it throw like the good old Entity Framework:

{% highlight C# %}

/* Startup.cs */

public void ConfigureServices(IServiceCollection services)
{
	services.AddDbContext<YourContext>(optionsBuilder =>
	{
		optionsBuilder
			.UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFQuerying;Trusted_Connection=True;")
			.ConfigureWarnings(warnings => warnings.Throw(RelationalEventId.QueryClientEvaluationWarning));
	});
}

/* Or in your context's OnConfiguring method */

protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
	optionsBuilder.ConfigureWarnings(warnings => warnings.Throw(RelationalEventId.QueryClientEvaluationWarning));
}

{% endhighlight %}

However, I found that disabling client evaluation made EF core so limited, that it became practically unusable. That's why I'm staying away from it for now and will re-evaluate it when version 3 is out to see if it's production-ready yet.