---
layout: post
title: Git&#58; commit with a UTC timestamp and ignore local timezone
---

When you git commit, Git automatically uses your system's local timezone by default, so for example if you're collaborating on a project from Sydney (UTC +10) and do a commit, your commit will look like this in git log for everyone:

{% highlight YAML %}
commit c00c61e48a5s69db5ee4976h825b521ha5bx9f5d
Author: Your Name <your@email.com>
Date:   Sun Sep 28 11:00:00 2014 +1000 # <-- your local time and timezone offset

Commit message here
{% endhighlight %}
	
If you find it rather unnecessary to include your local timezone in your commits, and would like to commit in UTC time for example, you have two options:

Changing your computer's timezone before doing a commit.
Using the `--date` commit option to override the author date used in the commit, like this:

{% highlight YAML %}
git commit --date=2014-09-28T01:00:00+0000
{% endhighlight %}

The first option is very inconvenient, changing system's timezone back and forth between UTC and local for commits is just silly, so let's forget about that. The second option however, seems to have potential, but manually inputting the current UTC time for each commit is cumbersome. We're programmers, there's gotta be a better way...

Bash commands and aliases to the rescue! we can use the date command to output the UTC time to an ISO 8601 format which is accepted by git commit's date option:

{% highlight YAML %}
git commit --date="$(date --utc +%Y-%m-%dT%H:%M:%S%z)"
{% endhighlight %}

We then alias it to a convenient git command like utccommit:

{% highlight YAML %}
git config --global alias.utccommit '!git commit --date="$(date --utc +%Y-%m-%dT%H:%M:%S%z)"'
{% endhighlight %}

Now whenever we want to commit with a UTC timestamp, we can just:

{% highlight YAML %}
git utccommit -m "Hey! I'm committing with a UTC timestamp!"
{% endhighlight %}