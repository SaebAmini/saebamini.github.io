---
layout: post
title: Create Your Own Visual Studio Code Snippets
---

Visual Studio [Code Snippets](https://msdn.microsoft.com/en-us/library/ms165392.aspx) are awesome productivity enhancers; I can only imagine how many millions of keystrokes I've saved over the years by making a habit out of using them.

Although a lot of common code you use daily might not be available out of the box, adding them yourself is very simple<!--more-->.

Here are some samples for creating `Console.ReadLine` & `Console.ReadKey` snippets:

Console.ReadLine:
{% highlight XML %}
<?xml version="1.0" encoding="utf-8" ?>
<CodeSnippets  xmlns="http://schemas.microsoft.com/VisualStudio/2005/CodeSnippet">
  <CodeSnippet Format="1.0.0">
    <Header>
      <Title>cr</Title>
      <Shortcut>cr</Shortcut>
      <Description>Code snippet for Console.ReadLine</Description>
      <Author>Community</Author>
      <SnippetTypes>
        <SnippetType>Expansion</SnippetType>
      </SnippetTypes>
    </Header>
    <Snippet>
      <Declarations>
        <Literal Editable="false">
          <ID>SystemConsole</ID>
          <Function>SimpleTypeName(global::System.Console)</Function>
        </Literal>
      </Declarations>
      <Code Language="csharp">
        <![CDATA[$SystemConsole$.ReadLine();$end$]]>
      </Code>
    </Snippet>
  </CodeSnippet>
</CodeSnippets>
{% endhighlight %}

Console.ReadKey:
{% highlight XML %}
<?xml version="1.0" encoding="utf-8" ?>
<CodeSnippets  xmlns="http://schemas.microsoft.com/VisualStudio/2005/CodeSnippet">
  <CodeSnippet Format="1.0.0">
    <Header>
      <Title>ck</Title>
      <Shortcut>ck</Shortcut>
      <Description>Code snippet for Console.ReadKey</Description>
      <Author>Community</Author>
      <SnippetTypes>
        <SnippetType>Expansion</SnippetType>
      </SnippetTypes>
    </Header>
    <Snippet>
      <Declarations>
        <Literal Editable="false">
          <ID>SystemConsole</ID>
          <Function>SimpleTypeName(global::System.Console)</Function>
        </Literal>
      </Declarations>
      <Code Language="csharp">
        <![CDATA[$SystemConsole$.ReadKey();$end$]]>
      </Code>
    </Snippet>
  </CodeSnippet>
</CodeSnippets>
{% endhighlight %}

You can save the above as `.snippet` files and then import them via `Tools > Code Snippet Manager... > Import...` and use them by typing `cr` or `ck` and hitting <kbd>TAB</kbd> twice.

So go ahead and create handy ones for things you find yourself typing all the time. You can refer to this [MSDN article for more details](https://docs.microsoft.com/en-us/visualstudio/ide/walkthrough-creating-a-code-snippet).