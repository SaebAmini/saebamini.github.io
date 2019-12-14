---
layout: post
title: Git&#58; Commit with a UTC Timestamp and Ignore Local Timezone
---

When you git commit, Git automatically uses your system's local timezone by default, so for example if you're collaborating on a project from Brisbane (UTC +10) and do a commit, your commit will look like this:

{% highlight YAML %}
commit c00c61e48a5s69db5ee4976h825b521ha5bx9f5d
Author: Your Name <your@email.com>
Date:   Sun Sep 28 11:00:00 2014 +1000 # <-- your local time and timezone offset

Commit message here
{% endhighlight %}
	
If you find it rather unnecessary to include your local timezone in your commits, and would like to commit in UTC time for example, you have two options<!--more-->:

1. Changing your computer's timezone before doing a commit.
2. Using the `--date` commit option to override the author date used in the commit, like this:

        git commit --date=2014-09-28T01:00:00+0000

The first option is obviously very inconvenient, changing the system's timezone back and forth between UTC and local for commits is just silly, so let's forget about that. The second option however, seems to have potential, but manually inputting the current UTC time for each commit is cumbersome. We're programmers, there's gotta be a better way...

Bash commands and aliases to the rescue! we can use the date command to output the UTC time to an ISO 8601 format which is accepted by git commit's date option:

{% highlight CFScript %}
git commit --date="$(date --utc +%Y-%m-%dT%H:%M:%S%z)"
{% endhighlight %}

We can then alias it to a convenient git command like `utccommit`:

{% highlight CFScript %}
git config --global alias.utccommit '!git commit --date="$(date --utc +%Y-%m-%dT%H:%M:%S%z)"'
{% endhighlight %}

Now whenever we want to commit with a UTC timestamp, we can just:

{% highlight CFScript %}
git utccommit -m "Hey! I'm committing with a UTC timestamp!"
{% endhighlight %}