---
layout: post
title: "Pooling Around with AI"
date: 2025-10-25 12:34:56 +0100
categories: pooling rust ai llm
---

This is me joining the crowd of self-proclaimed AI experts (thank you LinkedIn for showing me posts of people doing every obvious thing you can do with AI).

Yes, it can write code! Yes, it can understand text! It cannot though yet properly understad spacial contexts (try matches-puzzles, where you need to move only a few of them to represent highest number of stuff like that - and you're likely going to see interesting reasoning and some hallucinations), which is understandable as all learning was inferred only from text. Yes, it also can generate funny (sometimes) pictures & videos!

![Gemini: "avocado travelling on a plane"]({{ site.url }}/assets/2025-10-25-pooling-around/airplane-avocado.png)
(source: Gemini, promnt: "avocado travelling on a plane")

In this post I want to share two stories to make my point. First one about successfully using a popular LLM (not really important which specific one) to create meaningful and potentially useful piece of software - with clean expectations, instructions and supervision. Another one about using the same popular LLM for more ambiguos and way more open problem, and LLM optimizing expectations (like creating fake metrics instead of running the workload) instead of solving the problem, yet in a very energetic and confident way.

My point being: LLM is a multiplier (`>1.0`) for productivity for (1) moderately well defined problems and (2) under supervision of someone who can understand every single line of code it produces and why. Without these to conditions, there is a high risk of wasting time (even though pace of improvements of Plan Mode & Thinking Mode are mind blowing) and LLM being unproductive multiplier (`<1.0`).

If you don't really know what you're doing, with LLM you just produce ~~shit~~ ~~bugs~~ ~~technical debt~~ useless code faster. Every senior engineer knows - the best code is the one that haven't been even written. If this last sentence was shoking to you ~~and caused butthurt~~, then ~~stop reading and quickly post stupid comment about it~~ you might benefit from learning more about [TRIZ](https://pl.wikipedia.org/wiki/TRIZ). In TRIZ, the best system is the one does not even exist, yet it's function is being executed properly. If you think about, this is the ultimate efficiency, the thing does it's job and yet the thing does not even exist!

First story, successful implementation of a generic resource pool: [pooling-around](https://github.com/sergey-melnychuk/pooling-around).

Second story, trickery & deceit ended up as a useless pile of code: [streamforge](https://github.com/sergey-melnychuk/streamforge).

AI is here to stay, and there are even voices that say directly: "Time of writing code by hand is over" (unfortunately cannot attribute the quote properly, but you got the point). This feels like driving a car 100 years ago comparing to now. A century ago a car driver had deep understanding of the whole system, could (and often had to) fix little issues completely on their own, it took significant amout of time to master the craft and build the skill. And now, after 100 years of steady improvements, literally anyone can drive a car. A car now can just literally drive itself!

This is what is waiting for us - everyone will be able to explain to the machine what they want it to do, and the machine will understand and maybe even do things the right way. And you will not need years of education and decades of experience to build something relatively simple (say, drive to the office or to the mall), yet deep knowledge & skill will become niche (think race car drivers).

Software is done [eating the World](https://a16z.com/why-software-is-eating-the-world/), now [AI is eating it](https://www.businessinsider.com/software-ate-world-now-ai-eating-software-saas-anthropic-2026-2) and while [Free Lunch is long over](https://news.ycombinator.com/item?id=27649343), I believe AI is the enabler not the existential risk, [the New Electricity](https://www.gsb.stanford.edu/insights/andrew-ng-why-ai-new-electricity) if you like.

On a final note - it you're still not using LLM for software engineering (not only coding, but code/architecture review, tightening test coverage, discovering bugs & edge cases etc), then **START TODAY**!
