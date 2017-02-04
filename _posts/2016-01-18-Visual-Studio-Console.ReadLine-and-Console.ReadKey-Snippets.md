---
layout: post
title: Visual Studio Console.ReadLine & Console.ReadKey Snippets
---

Visual Studio [Code Snippets](https://msdn.microsoft.com/en-us/library/ms165392.aspx) are awesome productivity enhancers; I can only imagine how many millions of keystrokes I've saved over the years by making a habit out of using them.

However, not all common pieces of code are available as snippets out of the box. Good examples are `Console.ReadLine` & `Console.ReadKey`. Fortunately, creating them yourself is very easy, just save the following as `.snippet` files and then import them via `Tools > Code Snippet Manager... > Import...`:

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

Now you can use them by typing `cr` or `ck` for Console.ReadLine or Console.ReadKey respectively and hitting <kbd>TAB</kbd> twice.