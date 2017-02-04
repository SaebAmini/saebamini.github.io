---
layout: post
title: Modeling PowerToys for Visual Studio 2013
---

I rarely use tools that generate code, however, one that has become a fixed asset of my programming toolbox is Visual Studio's class designer. It's a great productivity tool that helps you quickly visualize and understand the class structure of projects, classes and class members. It's also great for presentation of code-base that does not come with a UI, e.g. a Class Library.

It also lets you quickly wire-frame your classes when doing [top-down design](https://en.wikipedia.org/wiki/Top-down_and_bottom-up_design), but it is limited in that aspect, for example it does not support [Auto-Implemented Properties](http://msdn.microsoft.com/en-us/library/bb384054.aspx), which I tend to almost always use in my Types, instead it blurts out a verbose Property declaration along with a backing field. Fortunately, almost all of these issues are fixed with the great [Modeling PowerToys](http://modeling.codeplex.com/) Visual Studio add-in by Lie which turns Class Designer into an amazing tool.

When I finally upgraded from my beloved Visual Studio 2010 to 2013, in the midst of all the horrors of VS 2013, I also found out that this add-in has not been updated to support later versions and the original author seems to be inactive, so I upgraded it myself and decided to put it here for other fellow developers who also happen to like the tool:

[Download Link](https://www.mediafire.com/?96gynhrxp6tbfih)

To install the add-in, extract the ZIP file contents to `%USERPROFILE%\Documents\Visual Studio 2013\Addins` and restart Visual Studio.

_Please note that I'm not the author of this add-in, I merely upgraded it for VS 2013._