---
layout: post
title: Integration Testing in Azure Functions with Dependency Injection
---

I'm a big proponent of integration and functional tests (or ["subcutaneous tests"](https://lostechies.com/jimmybogard/2010/08/25/an-effective-testing-strategy/) if there's a UI). Done efficiently and sparingly, they give you the biggest bang for your buck when it comes to confidence in the overall well-being of the majority of your system.

Dependency injection is ubiquitous these days, and [ASP.NET Core MVC's seamless support for integration testing](https://docs.microsoft.com/en-us/aspnet/core/test/integration-tests) via the [`Microsoft.AspNetCore.Mvc.Testing` NuGet package](https://www.nuget.org/packages/Microsoft.AspNetCore.Mvc.Testing) has made this kind of testing simple when using dependency injection. However, I found that when dealing with [Azure Function Apps](https://docs.microsoft.com/en-us/azure/azure-functions/functions-overview), a similar setup is not as effortless as installing a package and using a `WebApplicationFactory`, especially since Azure Function projects aren't set up for dependency injection out of the box. Fortunately, after a bit of digging under the hood of `Microsoft.AspNetCore.Mvc.Testing` and `Microsoft.AspNetCore.TestHost`, I've been able to create a similar testing experience which I'll go through below<!--more-->.

<div class="tip" markdown="1">
The setup isn't _identical_. `Microsoft.AspNetCore.Mvc.Testing` bootstraps an in-memory test server that you can communicate with using an HTTP client, however, that's part of ASP.NET Core while Azure Functions are built on top of the WebJobs SDK. While we can't have that identical setup with functions, we can still meet our goal of setting up high-level integration tests while leveraging existing dependency injection setup and settings from our "real" functions without any duplication. We do this by operating at the level after HTTP by just bootstrapping a test `IHost`.
</div>

## Sample Function App With Dependency Injection

Let's assume the below Azure Function app, with one HTTP endpoint that responds with the answer to life, the universe and everything:

{% highlight C# %}
public class SuperFunction
{
	readonly IHitchhikerGuideToTheGalaxy _hitchhikerGuideToTheGalaxy = new HitchhikerGuideToTheGalaxy();

	[FunctionName(nameof(AnswerToLifeTheUniverseAndEverything))]
	public IActionResult AnswerToLifeTheUniverseAndEverything(
		[HttpTrigger(AuthorizationLevel.Anonymous)] HttpRequest req)
	{
		return new OkObjectResult(_hitchhikerGuideToTheGalaxy.GetTheAnswerToLifeTheUniverseAndEverything());
	}
}

public interface IHitchhikerGuideToTheGalaxy
{
	int GetTheAnswerToLifeTheUniverseAndEverything();
}

public class HitchhikerGuideToTheGalaxy : IHitchhikerGuideToTheGalaxy
{
	readonly ISuperComputer _superComputer = new SuperComputer();

	public int GetTheAnswerToLifeTheUniverseAndEverything() => _superComputer.CalculateTheAnswerToLifeTheUniverseAndEverything();
}

public interface ISuperComputer
{
	int CalculateTheAnswerToLifeTheUniverseAndEverything();
}

public class SuperComputer : ISuperComputer
{
	public int CalculateTheAnswerToLifeTheUniverseAndEverything() => 42;
}
{% endhighlight %}

But we all know that [`new` is glue](https://ardalis.com/new-is-glue) for dependencies, so let's inject them. Support for dependency injection in Azure Functions has been added since Azure Functions 2.x. To register `IHitchhikerGuideToTheGalaxy` and `ISuperComputer` as services and inject them, we'll create a `Startup` class, similar to what we have with ASP.NET Core applications:

{% highlight C# %}
using Microsoft.Azure.Functions.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection;
using SuperFunctionApp;

[assembly: FunctionsStartup(typeof(Startup))]

namespace SuperFunctionApp
{
    public class Startup : FunctionsStartup
    {
        public override void Configure(IFunctionsHostBuilder builder)
        {
            builder.Services.AddTransient<IHitchhikerGuideToTheGalaxy, HitchhikerGuideToTheGalaxy>();
            builder.Services.AddTransient<ISuperComputer, SuperComputer>();
        }
    }
}
{% endhighlight %}

It has to inherit from `FunctionsStartup` and you also need to add a `FunctionsStartupAttribute` assembly attribute that points to the Startup class. Both of these types exist in the `Microsoft.Azure.Functions.Extensions` NuGet package which you need to install.

Now we can have some dependency injection goodness with the Function Host automatically injecting the dependencies for us:

{% highlight C# %}
public class SuperFunction
{
	readonly IHitchhikerGuideToTheGalaxy _hitchhikerGuideToTheGalaxy;

	public SuperFunction(IHitchhikerGuideToTheGalaxy hitchhikerGuideToTheGalaxy)
	{
		_hitchhikerGuideToTheGalaxy = hitchhikerGuideToTheGalaxy;
	}

	[FunctionName(nameof(AnswerToLifeTheUniverseAndEverything))]
	public IActionResult AnswerToLifeTheUniverseAndEverything(
		[HttpTrigger(AuthorizationLevel.Anonymous)] HttpRequest req)
	{
		return new OkObjectResult(_hitchhikerGuideToTheGalaxy.GetTheAnswerToLifeTheUniverseAndEverything());
	}
}

public interface IHitchhikerGuideToTheGalaxy
{
	int GetTheAnswerToLifeTheUniverseAndEverything();
}

public class HitchhikerGuideToTheGalaxy : IHitchhikerGuideToTheGalaxy
{
	readonly ISuperComputer _superComputer;

	public HitchhikerGuideToTheGalaxy(ISuperComputer superComputer)
	{
		_superComputer = superComputer;
	}

	public int GetTheAnswerToLifeTheUniverseAndEverything() => _superComputer.CalculateTheAnswerToLifeTheUniverseAndEverything();
}

public interface ISuperComputer
{
	int CalculateTheAnswerToLifeTheUniverseAndEverything();
}

public class SuperComputer : ISuperComputer
{
	public int CalculateTheAnswerToLifeTheUniverseAndEverything() => 42;
}
{% endhighlight %}

But how do we now properly test that our function correctly returns what we know is the obvious answer to life, the universe and everything? testing the various bits separately would cross into the realm of unit-testing, while duplicating our dependency injection setup is far from ideal; we want to keep and use all our existing DI configuration from our `Startup` in our tests.

## Setting Up An Azure Function Test Host

The secret to using our existing `Startup` configuration and DI for integration testing lies in bootstrapping a [.NET Core Generic Host](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host) via the `ConfigureWebJobs` extension method: 

{% highlight C# %}
var startup = new Startup();
var host = new HostBuilder()
	.ConfigureWebJobs(startup.Configure)
	.Build();
{% endhighlight %}

<div class="tip" markdown="1">
The use of `ConfigureWebJobs` might seem a bit strange, but since Azure Functions are built on top of the Web Jobs SDK, there is significant shared and compatible API surface. Here we are using the overload that takes an `Action<IWebJobsBuilder> configure` argument. Our `Startup` class inherits from `FunctionsStartup` which inherits from `IWebJobsStartup`, [providing a compatible Configure method that calls our `Configure(IFunctionsHostBuilder builder)` method](https://github.com/Azure/azure-functions-dotnet-extensions/blob/master/src/Extensions/DependencyInjection/FunctionsStartup.cs)!
</div>

Putting the above together, we can write an integration test, that initializes our HTTP Function reusing all the dependency injection and settings from our "real" code inside a test Host, and tests whether it correctly returns the true answer to life, the universe and everything:

{% highlight C# %}
public class SuperFunctionTests
{
	readonly SuperFunction _sut;

	public SuperFunctionTests()
	{
		var startup = new Startup();
		var host = new HostBuilder()
			.ConfigureWebJobs(startup.Configure)
			.Build();

		_sut = new SuperFunction(host.Services.GetRequiredService<IHitchhikerGuideToTheGalaxy>());
	}

	[Fact]
	public void Test()
	{
		// arrange
		var req = new DefaultHttpRequest(new DefaultHttpContext());

		// act
		var result = (OkObjectResult)_sut.AnswerToLifeTheUniverseAndEverything(req);

		// assert
		Assert.Equal(42, result.Value);
	}
}
{% endhighlight %}

If you'd like to quickly try it for yourself, I have shared my preferred way to set up Azure Function projects as a [.NET Core Template on GitHub](https://github.com/SaebAmini/Saeb.FunctionApp). The template includes a test set up with the above approach, plus using the familiar `appsettings.json` file for settings instead of `local.settings.json` and Environment Variables, to help with ease and simplicity of deployment, as well as a build defined via Azure YAML Pipelines.
