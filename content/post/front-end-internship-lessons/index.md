---
title: "The Non-Code Half of Engineering: Lessons from a Front-End Internship"
description: "My first real role was a hybrid: half building websites, half sitting between clients and the technical team. The second half taught me more about engineering than the first."
slug: "front-end-internship-lessons"
date: 2024-08-01
categories:
    - Notes
tags:
    - Career
    - Front-End
---

Before I moved toward machine learning, my first real role was a front-end
development internship at **Mandala Infosys** in Kathmandu, from April to July
2024. On paper it was an HTML-and-CSS job. In practice it was a hybrid — half
building, half translating between what clients *wanted* and what the technical
team could *build* — and the translating half is where I learned the most.

## The build half

The straightforward part was writing the front end. I built responsive interfaces
in HTML, CSS, and JavaScript: semantic navigation components, landing-page layouts
with formatted and optimised imagery, and form pages. It was a normal agile setup —
sprint planning, code reviews, iterating on feedback — and it's where I got my first
real taste of shipping something other people would actually use, on a deadline,
with someone reviewing my work.

That was valuable. But it wasn't the part that changed how I think.

## The half that mattered

The more unusual side of the role put me between the client and the developers. I'd
sit with customers to **elicit what they actually wanted** from a website — which,
early on, I learned is almost never what they *first say* they want. So I started
building small **reference builds**: quick, concrete mock-ups I could demo, because
a vague request turns specific fast the moment someone can point at a real screen
and say "not that — this."

From there I'd draft **solution options** for the technical team to weigh, and this
is where the actual lesson lived: **feature prioritisation**. Clients want
everything. Time and budget don't allow everything. My job became walking
non-technical people through the *opportunity cost* of each choice — "if we build
this, here's what it costs, and here's what it pushes out of scope" — so they could
make an informed trade-off instead of a wish list.

Saying "yes, and here's what that costs you" turned out to be far more useful than
saying "yes" to everything.

## Why this connects to reliability

At the time I didn't see the thread. I do now. The work I care about today — ML
reliability, keeping systems trustworthy in production — is *also* mostly about
honest trade-offs. When I decide [not to auto-retrain a drifted model](/ml-reliability-pipeline/),
or [keep a known failure visible instead of hiding it behind a nicer metric](/p/vocabulary-not-mechanism/),
or [refuse to cite a number I haven't measured](/berlin-transit/) — that's the same
instinct I first practised in a small office in Kathmandu, explaining to a client
why we shouldn't build the thing they asked for.

Build the thing. But stay honest about what every decision trades away. That's the
non-code half of engineering, and it's the half I'd bet on.