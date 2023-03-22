---
layout: post
title: A Developer Culture
subtitle: A comparison of we Indians and European developers
---

Disclaimer : I speak for myself and this is purely based on my personal experiences.

As a developer, last few years have been an amazing ride. I have seen wide varieties of development culture and practices and one thing I can say for sure is that nothing works perfect. It is interesting to see how developers from different backgrounds practice and here is a brief comparison based on my experience with various developers.

The wide gap that i have observed between we Indians and europeans is sticking to rules and books. Quite obvious when we see how we behave on the roads and how traffic works in european cities. We Indians are more flexible, chaotic, but we get things done. These europeans on the other hand are more orderly, sticking to what the rulebook says.

We definitely do not use all our learnings i school at the workplace. We study (hard or hardly), get some marks, crack interviews, and then work the way we want or the way stack overflow suggests. I have seen very less people in India, who actually cared for design patterns for example. Or who actually cared for how test cases should be written, or the logging practices. We studied Effective Java to crack some interview, and then forgot everything.

On the bright side, we get things done, on time!

Compared to how we do things, I have seen such wide difference in workplace practices here in Berlin. 

## Coding :

Extremely strict adherence to coding styles. Code reviewers strictly enforce the best practices followed. They do not leave any spectrum of coding. The reviews cover everything that can be imagined. Unused import statements, wildcard imports, non-final declarations, proper exception types, naming conventions, tab spaces, anti patterns like log-and throw, static vs non-static, package structures, buffered output writer vs print writer etc. Going through such a review simply refreshes everything from Effective Java. In the end you get an elegant, beautiful code snippet.

On the downside, each review can run for weeks. Coding is something very personal to many passionate developers. And in situations where we have passionate but opinionated developers, they stick to what they believe is correct (e.g., streams vs for-each or own implementation vs library), causing friction in the team. Sometimes it is very difficult to get everyone on the same page. In India, one would hardly argue passionately for streams vs for-each as long as the work is done. We consider it as not earth shattering thing. But not so much here. Of course not all europeans are like these.

## Test cases :

I had never seen such elegant test cases written in India that they write here. We used to write test cases based on what we felt about the priority corner cases. If we know certain logic in the flow is critical, we would write a test case for that. Many people won’t even bother about writing test cases for very trivial things.

Here, nothing is considered trivial. Every single line is unit tested unless there is something that is really unavoidable. Like mocking static methods as an example. Half of the code bases that I worked in India, never had any integration tests. Many times, integration tests were left as something to be done by the QA guys and developers just trusted them that they are tested. Here, the reviewers would probably not accept if any integration test is left out.

## Design :

The design runs for months. Literally! And it many times becomes frustrating. I would say it is more due to the opinionated developers who are too rigid to accept any other alternate view point. MySQL vs postgres, stored procedure or not, events vs queues etc. Sometimes it is enlightening as you get to know things that others know in depth. And sometimes it is frustrating as hell as there is nothing in the world called the perfect design, and arguing over things again and again takes you eventually nowhere.

What I do like about  this is, they are extremely data driven. Unlike in india where many of the decisions were being taken on guess work, here every single thing is documented and analyzed. For example, if you are going to use Postgres, you will come up with extremely detailed analysis of how much time and space it will take, pros, cons everything! I never had such an encounter in India. For sure, these are very enlightening, in a way making you a domain expert. On the other hand, too much analysis leads to paralysis and the design phase just keeps on running on and on.

## Sprint planning and tasks :

These people won’t touch a task unless all, each and every possible specifications are mentioned. In my previous experiences, we generally used to start coding with certain ’seemingly clear’ requirements and it was not rare to change the requirements time and again. Here, each task is properly defined (as opposed to usually 1 liners in india) with specifications and acceptance criteria and expected timelines. Separate ‘grooming’ sessions are done to ‘groom’ each and every task. The expected timelines are also taken on consensus where everyone votes his idea of expected man-hours in fibonacci numbers. A sprint planning is not complete without demos of completed tasks and retrospectives where you highlight what all things went good, and what all things needs improvement. The retrospective doesn’t not just include what all things couldn’t be completed, but it is more than that to include any work related issue: too many meetings, dependencies, leaves, anything that impacts workplace.

There is nothing special about this and this is how it is supposed to work. However, in India, the sprint planning meetings were mostly confined to backlog clearing, assigning of tasks to members and practically there used to be no retrospectives apart from noting down what all tasks were completed and what all couldn’t be completed.

### Conclusion 

I wish we had a middle ground. I setup two indian startups backend on AWS over a matter of weeks. I cannot imagine happening that with an European development culture where the design phase runs for months. At the same time, I wish we were more adherent to coding practices and patterns in our development jobs instead of just learning them during interviews and unlearn them soon after. And as far as sprint planning is concerned, I can do just fine without the damn retrospectives.

There is no point comparing qualitative aspects like work life balance as they are very very different and everybody knows that it is fucked up in India.
