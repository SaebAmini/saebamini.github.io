---
layout: post
title: How to Version Your .NET NuGet Packages, Libraries and Assemblies + Azure YAML Pipelines Example using .NET Core CLI
---

I often see arbitrary patterns used for versioning packages and assemblies, especially for internal projects, with the only common goal being that the versions are increased with each release. There is rarely a distinction between NuGet package `Version`, `AssemblyVersion`, `FileVersion` and `InformationalVersion` and they're commonly all set to the same value, if set at all. That's not to blame anyone, having all these different ways to set a version is darn confusing.

However, if you plan to release a public NuGet package, or an internal package that could be heavily used, you'll be making your life, and your users' lives a lot easier, and avoid creating a "dependency hell", if you distinguish between the above versions and follow a meaningful versioning strategy like [Semantic Versioning](https://semver.org/).

If you haven't heard about Semantic Versioning, here's the TL;DR:

- It solves the problems of _version lock_, the inability to upgrade a package with minor changes and bug fixes without having to release new versions of every dependent package, and _version promiscuity_, the inability to safely upgrade a package with breaking changes without automatically breaking dependent packages.

- The versioning scheme is `{major}.{minor}.{patch}-{tag}+{buildmetadata}`. `{major}`, `{minor}` and `{patch}` are numeric, while `{tag}` and `{buildmetadata}` can be alphanumeric.

- Bug fixes increment the `{patch}` number.

- Backward compatible additions/changes increment the `{minor}` number.

- Only breaking changes increment the `{major}` number. You can refer to [CoreFX's rules for classifying breaking changes](https://github.com/dotnet/corefx/blob/master/Documentation/coding-guidelines/breaking-change-rules.md) as a guideline.

- The `-{tag}` and `+{buildmetadata}` parts are optional. `-{tag}` is used to denote pre-release unstable versions and so is used less frequently, although I recommend always including the `+{buildmetadata}` to provide valuable information – more on this below.

So how do you follow and apply that in the .NET ecosystem?

I've recently released a NuGet package, called [EF.Auditor](https://github.com/SaebAmini/EF.Auditor) – a simple and lightweight auditing library for Entity Framework Core that supports Domain Driven Design. As part of releasing this package, I refreshed my knowledge around versioning, the multitude of version attributes and researched versioning strategies used by Microsoft and popular packages to arrive at the sweet spot of versioning packages and assemblies. I'll go through the highlights below. If you'd like to skip the details and just see the versioning strategy, jump to the ["Putting it all together"](#putting-it-all-together) part.

## What the heck are NuGet package `Version`, `AssemblyVersion`, `FileVersion`, `InformationalVersion`

If you're confused by all the different versions, you're not alone. However, while they might all look the same, they have quite different characteristics and effects.

### NuGet package version

This is the version in your package's `nuspec` file in the `<version>` element.

The NuGet package version is displayed on [NuGet.org](https://www.nuget.org/packages/ef.auditor), the Visual Studio NuGet package manager and is the version number users will commonly see, and they'll refer to it when they talk about the version of a library they're using. The NuGet package version is used by NuGet and has _no effect_ on runtime behavior.

Since this version is the most visible version to developers, it's a good idea to update it using Semantic Versioning. SemVer indicates the significance of changes between release and helps developers make an informed decision when choosing what version to use. For example, going from 1.0 to 2.0 indicates that there are potentially breaking changes.

However, since Visual Studio < 2017 and NuGet client < 4.3.0 do not support SemVer 2.0.0, I suggest sticking with SemVer 1.0.0 here for maximum compatibility.

### AssemblyVersion

Commonly defined in the `csproj` file, or `AssemblyInfo.cs` in old project formats, the assembly version is what the CLR uses at runtime to select which version of an assembly to load.

The .NET Framework CLR demands an exact match to load a strong named assembly. For example, `Libary1, Version=1.0.0.0` was compiled with a reference to `Newtonsoft.Json, Version=11.0.0.0`. The .NET Framework will only load that exact version `11.0.0.0`. To load a different version at runtime, a binding redirect must be added to the .NET application's config file.

Strong naming combined with assembly version enables strict assembly version loading. While strong naming a library has a number of benefits, it often results in runtime exceptions that an assembly can't be found and requires binding redirects to be fixed. .NET Core assembly loading has been relaxed, and the .NET Core CLR will automatically load assemblies at runtime with a higher version, but the .NET Framework CLR needs the entire AssemblyVersion has to be an exact match in order for an assembly to be loaded.

For maximum binary compatibility with the .NET Framework CLR and strong-naming scenarios, only sync the major version of this attribute to match the major part of your semantic version and leave the other parts as `0`. This is the strategy followed by .NET Framework BCL, CoreFX and many other popular packages like Entity Framework Core and Newtonsoft.Json.

Providing a `*` in place of the numbers is likely one of the worst things to do as it makes compiler set numbers based on [some crazy rules](https://docs.microsoft.com/en-us/dotnet/api/system.reflection.assemblyversionattribute) and you'd need a calculator to get any valuable information out of those numbers. It also would cause an error in .NET Core projects as they are [deterministic](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/compiler-options/deterministic-compiler-option) by default.

Note that although Assembly Version follows a similar `{Major}.{Minor}.{BuildNumber}.{Revision}` schema, it has significant limitations so you won't be able to apply SemVer here anyway:

- It only supports numbers for all parts.
- Each number [can only go up to 65534](https://blogs.msdn.microsoft.com/msbuild/2007/01/03/why-are-build-numbers-limited-to-65535/), so you wouldn't be able to, for example, have a useful full date such as 190920 included in your revision. Using an auto-incremented build number from your CI here [would also get you only so far](https://stackoverflow.com/q/1188284/68080).

### FileVersion

Commonly defined in the `csproj` file, or `AssemblyInfo.cs` in old project formats, the assembly file version is used to display a file version in Windows and has _no effect_ on runtime behavior. Setting this version is optional. It's visible in the File Properties dialog in Windows Explorer:

![File Version](/images/posts/versioning/file-version.png)

It's also the version displayed in the Windows Explorer tooltip when you hover over an assembly:

![File Version](/images/posts/versioning/file-version-tooltip.png)


File Version follows the same versioning schema as Assembly Version and has the same limitations, so you can't use this one for proper SemVer versioning either. But for consistency and preventing user confusion, I recommend keeping the `{Major}.{Minor}.{BuildNumber}` parts of this attribute in sync with the `{major}.{minor}.{patch}` from your semantic version. Changing this attribute frequently won't create the same issues as changing Assembly Version explained above, as this attribute is never used by .NET.

### InformationalVersion

Commonly defined in the `csproj` file, or `AssemblyInfo.cs` in old project formats, the assembly informational version is similar to File Version in that setting it is optional, it has _no effect_ on runtime behavior and it shows up in File Properties dialog in Windows Explorer as `Product version`:

![Product Version](/images/posts/versioning/informational-version.png)

The major difference is this version can be _any_ string and doesn't have the same limitations as Assembly Version and File Version, which makes it perfect for being set to your full SemVer 2.0.0 version.

The `InformationalVersion` assembly attribute could be read at runtime and used to identify the exact version of your package or application, for example in logging:

{% highlight C# %}
string semanticVersion = assembly.GetCustomAttribute<AssemblyInformationalVersionAttribute>().InformationalVersion;
{% endhighlight %}

## Putting it all together

Combining the above information and after trying many different approaches through the years, I've found the below strategy to be at the sweet spot of versioning NuGet packages and .NET assemblies:

- Set your NuGet package version to the `{major}.{minor}.{patch}-{tag}` part of your Semantic Version
- Follow [the SemVer specs](https://semver.org/#semantic-versioning-specification-semver) for bumping the `{major}`, `{minor}` and `{patch}` parts
- Set your `AssemblyVersion` to `{major}.0.0.0`
- Set your `FileVersion` to `{major}.{minor}.{patch}.0`
- Set your `InformationalVersion` to the full Semantic Version of `{major}.{minor}.{patch}-{tag}+{buildmetadata}`
  - For `{buildmetadata}`, use the pattern of `{BuildDate:yyyyMMdd}{BuildRevision}.{commitId}`. This provides you with super-useful information at a glance:
    - When the build was done
    - The exact commit in your source control history the build was from, which helps you quickly check out that version for troubleshooting
- Only include the `-{tag}` part if you're releasing an unstable pre-release version, e.g. `-alpha`

How you implement the strategy would depend on your platform, tooling and build process. While you could go with tools like GitVersion, my preferred method these days is a simple source-controlled way of managing the version through [Azure YAML Pipelines](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema), close to where the rest of my build definition is.

### Azure YAML Pipelines example

Here's EF.Auditor's versioning done through Azure YAML Pipelines:


{% highlight YAML %}
name: $(Date:yyyyMMdd)$(Rev:r)

pool:
  vmImage: 'Ubuntu 16.04'

variables:
  buildConfiguration: 'Release'
  versionMajor: 1
  versionMinor: 1
  versionPatch: 5

steps:
- powershell: |
    $mainVersion = "$(versionMajor).$(versionMinor).$(versionPatch)"
    $commitId = "$(Build.SourceVersion)".Substring(0,7)
    Write-Host "##vso[task.setvariable variable=mainVersion]$mainVersion"
    Write-Host "##vso[task.setvariable variable=semanticVersion]$mainVersion+$(Build.BuildNumber).$commitId"
    Write-Host "##vso[task.setvariable variable=assemblyVersion]$(versionMajor).0.0"
  name: SetVersionVariables

- powershell: |
    dotnet pack --configuration Release /p:AssemblyVersion='$(assemblyVersion)' /p:FileVersion='$(mainVersion)' /p:InformationalVersion='$(semanticVersion)' /p:Version='$(mainVersion)' --output '$(Build.ArtifactStagingDirectory)'
  name: Pack

- task: PublishBuildArtifacts@1
{% endhighlight %}

I've simplified the pipeline to the bits relevant to versioning; you can [see the full pipeline definition on GitHub](https://github.com/SaebAmini/EF.Auditor/blob/master/azure-pipelines.yml).