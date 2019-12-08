---
layout: post
title: Code Reviews&#58; An Accurate Window into a Team's Health
---

Pull requests and code reviews are one of the most powerful software quality tools we have available in our tool-belt.

Even when flying (han) solo on a project, I tend to create feature branches and PRs and review my own code after a short break; I usually find a few things I can improve when doing that, with the added benefit of being able to switch easily to other work if needed.

In teams, the benefits are many and more significant: extra pair of eyes are always better in spotting improvement opportunities and catching mistakes, they consistently raise the quality of the project in all areas to the best of the team's knowledge rather than individuals, they reduce the project risk by sharing knowledge and breaking down information silos while upskilling everyone. To a much _much_ lesser degree of importance, they are about guarding the project against YOLO pushes of crap code – that's more relevant for public, open-source projects; if that's your main reason for using pull requests and code reviews, that's a sign of bigger root problems in the team.

However, this post isn't about those more commonly known benefits; it's about how code reviews are one of the best places to observe and gauge a team's health. I recently thought about this as I was reading the excellent book called ["The Five Dysfunctions of a Team"](https://en.wikipedia.org/wiki/The_Five_Dysfunctions_of_a_Team), recommended to me by my dear colleague [Mehdi Khalili](https://www.mehdi-khalili.com/).

In the book, Patrick Lencioni talks about the following five core team dysfunctions which build on top of each other. I'll go through them and note how they can be observed in PRs and code reviews.

![The Five Dysfunctions of a Team](/images/posts/code-reviews/dysfunctions-pyramid.png)

## Absence of trust

The most fundamental dysfunction in a team is a lack of trust, which is essentially the team members not willing to be vulnerable and open to each other. This results in a huge waste of time and energy as team members invest them in defensive behaviours, and are reluctant to ask for help from each other.

Putting your code up in front of colleagues to review and give feedback on is essentially an act of vulnerability and people opposing the idea of code reviews is a strong sign of this dysfunction.

Tackling this issue has to start with the leaders. Team and tech leads need to keep their minds open and look for opportunities to learn from and ask for help from junior people. You also have to cultivate a safe culture where mistakes aren't frowned upon, but are seen as an opportunity to learn and improve. Otherwise, they keep happening and growing silently until they lead to big failures.


## Fear of conflict

> When there is trust, conflict becomes nothing but the pursuit of truth, an attempt to find the best possible answer. – Patrick Lencioni

Healthy, constructive conflict and discussion within a team is good thing (tension isn't; don't confuse the two). Diversity of thoughts leads to improvements and better outcomes.

If there is trust, but code reviews are going through with little or no comments or discussions, that's a strong sign that the team has a fear of conflict and wants to maintain an artificial harmony. At this point, pull requests and code reviews become merely red tape.

To tackle this, encourage everyone to give feedback or even ask questions. Encourage opinions, but have an open mind and hold them loosely. However, be careful to not fall into the trap of [bike-shedding on trivial issues](https://en.wikipedia.org/wiki/Law_of_triviality).

Having code review etiquette also definitely helps. Be nice and show how something could be improved rather than just criticising. Call out the positive things you like. Don't ever make feedback personal; it should always be about the code and don't forget that people are not their code!


## Lack of commitment

This happens when people haven't had a chance to provide their input or be heard. When people don't weigh in on something, they don't buy in to it. This is not about seeking consensus (although it's good if you have that); it's more about making sure people are heard.

If there is no fear of conflict, but people are reluctant to perform code reviews or give their vote to them, that is a sign of either this dysfunction or the next one.

To solve this, team members (including seniors) should regularly bounce ideas off each other and ask for other team members' opinions about how to approach a particular problem, even if they already have in mind what they think is a solid approach.


## Avoidance of accountability

In a well-functioning team, it's the responsibility of each team member to hold one another accountable, and accept it when others hold them accountable. Otherwise, they'll end up with low standards.

This happens when people have a lack of clarity on the work being done. They don't understand it properly or they don't know why it's being done.

The signs of this dysfunction in code reviews is very similar to the previous one: people are either reluctant to perform code reviews, or they make trivial comments but then leave them in a pending state without approving or voting otherwise.

To solve this, have clear communication around expectations and ensure the team understands why something is being done. Abiding by the user story template of "As a [persona], I want [feature], so that [outcome]" and having clear and detailed acceptance criteria helps here; if you have trouble with these, that's usually because of poor connection and communication with the business or end-users.


## Inattention to results

This is often because individual contributions and goals are prioritized over the team's goal and collective business outcomes, which is usually a result of big egos, politics and individual KPIs.

If the previous dysfunctions don't exist, but you find that people have to be constantly pushed and nudged to review pending pull requests, and it usually takes a long time before they happen, that's a strong sign of this dysfunction, as people are prioritizing their WIP individual contributions over shipping work that's nearly done.

To tackle this, you need to move towards a team-oriented culture and thinking. The whole of a well-functioning team is greater than the sum of its parts.

Goals, recognitions and rewards should primarily be for teams rather than individuals. However, it's _crucial_ that teams are small enough that each individual feels the effects of their contributions towards the collective outcome, otherwise, the motivating effects are lost.

&nbsp;

Note that code reviews don't fix these dysfunctions by themselves; but they are a great place where the dysfunctions are exhibited, letting you observe and identify them – a critical first step in fixing them. This, in addition to the benefits mentioned in the beginning, makes code reviews the single highest impact practice you can have to help you improve your product _and_ your team, both from a technical and people perspective.