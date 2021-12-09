---
title: "Keeping a journal"
date: 2021-12-09T16:40:45Z
tags:
- self-improvement
categories:
- Tips and Tricks
summary: How keeping a journal can improve your productivity.
---

In this article I'll describe how taking notes improved my productivity.

Many years ago, upon starting a new job, I decided to keep a journal; generally when you start a new
job you spend some time doing training and learning about your surroundings, which can be quite
overwhelming. And you get exposed to all this new knowledge in just a couple of weeks!

Keeping a journal seemed the best strategy for me to not get lost and be productive as quickly as
possible.

My strategy for journaling was (and still is) quite simple: I jot down notes on a text file divided
into sections. Every day I create a new section and every task gets its section.

It looks roughly like this:

```org
* <2021-11-01 Mon>

Another training day!

** AWS training

Concepts:

- AWS console: https://aws.amazon.com/console/
- ec2 (virtual server)
- NLB (network load balancer)

** CI and Deploys

- Jenkins at: http://ci.example.com/
```

This snippet shows how I would take notes in a day of training where I got to learn about AWS and
how this company does Continuous Integration and deployments.

Let's see another example: fast-forward six months, the training are over and now Jira is your worst
enemy (or friend!). This is how my notes would look like:

```org
* <2021-12-09 Thu>

** PROJECT-1234 Provision new reverse proxies

REMEMBER! need to change the listening port to 8080!
(it's the new standard)

Also, try to schedule a pairing session with John before sending
your code for review.
```

This snippet shows my notes for a hypothetical Jira ticket (`PROJECT-1234`) with two reminders.

To summarize what I've shown so far, my strategy for note-taking is quite simple: I try to capture
short thoughts, reminders and notes grouped by context (which can be a task, a training session, a
meeting). In some cases I copy/paste the output of a command, or a piece of configuration file, or
an error message.

The general idea is: if one day I need to work on a task similar to one that I've worked on in the
past, or I encounter a problem that I've already solved, can I refer to my notes to find sufficient
information? If I take a week of time off while working on something, will I be able to resume right
where I left?

I use the same journaling strategy for personal stuff: installing Home Assistant on your Raspberry
Pi? Notes! Trying a new command-line tool? Notes! Switched to a new phone contract? Notes!

So far I did not mention anything about specific software or technology because it's not really
important; I found that solutions with the least friction work best for me, but it's a personal
choice.

For my note-taking I use Emacs' Org Mode, at 5% of its potential: I don't use any key-binding other
than "insert date" and the TAB key to expand a couple of macros; the only other main feature I use
is the syntax highlighting, that makes my notes more readable and helps in distinguishing one
section from the others. I could be using Markdown with Vim, or Notepad, or any other text editor,
and my workflow would be pretty much the same. Actually I don't recommend using Org Mode at all
unless you're already using Emacs: writing your notes should be as frictionless as possible.

With that said I'll mention a couple of software that could help get you started with note-taking:

- [jrnl](https://jrnl.sh/en/stable/overview/): `jrnl` is a simple journal application for the
  command line.
- [Obsidian](https://obsidian.md/): a knowledge base application that uses plain text files written
  in Markdown.
- [Evernote](https://evernote.com/): a paid software (and service) with multi-platform clients,
  cloud storage and all the features you would expect from a commercial product.
