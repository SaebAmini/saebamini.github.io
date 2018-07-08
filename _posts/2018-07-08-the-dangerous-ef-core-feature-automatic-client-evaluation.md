---
layout: post
title: The dangerous EF Core feature&#58; automatic client evaluation
---

Recently when going through our shiny new ASP.NET Core application's [Seq](https://getseq.net/) logs, I accidentally spotted a few entried like this:

> The LINQ expression 'foo' could not be translated and will be evaluated locally.

Odd.

I dug around in the code and found the responsible queries. Some of them were quite complex with many joins and groupings, while some of the other ones were very simple stuff, like `someStringField.Contains("bar", StringComparison.OrdinalIgnoreCase)`.

You may have spotted the problem right away. `StringComparison.OrdinalIgnoreCase` is a .NET concept. It doesn't translate to SQL and you can't blame EF Core for that. As a matter of fact if you run the same query in full-blown Entity Framework, you'll get a `NotSupportedException` telling you it can't convert your perdicate to a SQL expression. And that's a _good_ thing! because it prompts you to _review_ your query, and if you _really_ want to have a predicate in your query that only makes sense in the CLR world, _you_ can decide if doing a `ToList()` at some point in your `IQueryable` to pull down the results of your query from the database into memory makes sense. And when you have your results in memory you can continue with your query shenanigans without having to worry if the rest of your query is translatable to SQL. Or you may decide that you don't need that `StringComparison.OrdinalIgnoreCase`, because your database collation is case-insensitive anyway.

The point is, by default, _you_ are in control and can make explicit decisions based on your circumstance.

Not anymore in Entity Framework Core. Apparently there's this concept of [mixed client/server evaluation in EF Core](https://docs.microsoft.com/en-us/ef/core/querying/client-eval). What it effectively does is if you put stuff in an `IQueryable` LINQ query that can't be translated to SQL or a query in your underlying database, it tries to magically make it work for you, by taking the untranslatable bits out and running them locally! and it's enabled by _default_!

That's a huge and extremely dangerous behavior change compared to the full Entity Framework. Consider this familiar entity:

{% highlight C# %}

public class Person
{
	public string FirstName { get; private set; }
	public string LastName { get; private set; }
    public List<Address> Addresses { get; private set; }
    public List<Order> Orders { get; private set; }
	
	/* 
	
	more properties and child collections
	
	*/
}

{% endhighlight %}

And imagine someone writing an EF query like this:

{% highlight C# %}

	var results = dbContext.Persons
		.Include(p => p.Addresses)
		.Include(p => p.Orders)
		.Where(p => p.LastName.Equals("Amini", StringComparison.OrdinalIgnoreCase))
		.ToList();

{% endhighlight %}

EF Core can't translate `p.LastName.Equals("Amini", StringComparison.OrdinalIgnoreCase)` into a query that can be run on the database, so what it does is, it pulls down the _whole_ Persons table, as well as the _whole_ Orders and Addresses tables from the database into the memory and _then_ runs the `.Where(p => p.LastName.Equals("Amini", StringComparison.OrdinalIgnoreCase))` filter on the results. 🤦

It's not hard to imagine the performance repercussions of that on any real-sized application with a significant number of users. It can easily bring down applications to their knees. Compare that to the full Entity Framework where you get an exception. The fact that this hugely different behavior is the default is _mind blowing_!

You could argue that it's the developer's fault for including something like `StringComparison.OrdinalIgnoreCase` in the IQueryable prediate. While that's true, you can't _blame_ people newer to EF for expecting something like that to magically work. Especially since it does! for more senior devs it can easily sneak past unnoticed when context switching between LINQ-to-entities and LINQ-to-objects.

Also, having untranslatable things like `StringComparison.OrdinalIgnoreCase` in your query isn't the only culprit that results in client evaluation. If you [have](https://github.com/aspnet/EntityFrameworkCore/issues/6245) too many [joins](https://stackoverflow.com/questions/45237492/ef-core-could-not-be-translated-and-will-be-evaluated-locally) or [groupings](https://github.com/aspnet/EntityFrameworkCore/issues/12255), the query can become too complex for EF Core and make it fall back to local evaluation.

So you probably want to keep an eye on your logs if your complex EF Core queries are magically working without a hitch as you may be getting client evaluation. Or better yet, if you don't want that unnecessary additional cognitive overhead, disable it altogether and make it throw like the good old Entity Framework:

{% highlight C# %}

/* Startup.cs */

protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
	optionsBuilder
		.UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFQuerying;Trusted_Connection=True;")
		.ConfigureWarnings(warnings => warnings.Throw(RelationalEventId.QueryClientEvaluationWarning));
}


{% endhighlight %}


I believe Languages and frameworks should always make it harder to make mistakes, especially ones like this with potentially devastating consequences, so I strongly disagree with this new default behavior and will be keeping it disabled.