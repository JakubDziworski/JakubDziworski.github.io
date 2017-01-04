---
layout: post
title: Applied Akka Patterns - Book Review
categories: [scala,akka,book]
tags: [scala,book,applied,akka,patterns]
fullview: true
comments: true
---


I recently watched [Wade Waldron's talk "Domain Driven Design and Onion Architecture in Scala"](https://youtu.be/MnNeDXg3Qao) 
which I found really great. What stood out was Wade's gift for explaining confusing topics
 in a way that anyone could understand. 
 
 Few days later I googled dude's name and it turned out him and Michael Nash are 
about to release a book called "Applied Akka Patterns". I skimmed through the table of contents 
and was initially going to read only one chapter, but damn, this book turned out to be excellent and I had to read it cover to cover.

## So many questions answered

When I first started learning Akka I had so many questions and there was no one to answer.

Should entire system be based on actor model or can I just use it parts of it?
How to deal with blocking operations? When to use futures and when to use actors?
How does DDD fit into akka? How to monitor and find bottlenecks in akka based system? 
Which operations deserve separate dispatcher? "Tell don't ask" - when should I use ask then? 
Which supervision strategies are useful in which scenarios?
I found answer to those questions on my long and painful path reading other people's code and 
finding fragmented information here and there.

The book answers all those questions and many many more. It amazes me how much useful content is packed into such a small volume (200 pages).
Some books leave you with more questions than you had had before you grabbed it - not this one. There were numerous times when I was reading a paragraph and thought to myself "Oh, that's fine but what about...?" and then
the answer was found right there on the next page. It almost feels like the authors took some beginner akka programmer, asked
to read the chapter and write all the questions down.
I also like that the book is very pragmatic - the theory is compressed to absolute minimum and almost all statements are backed up by practical examples - even chapter regarding DDD.

The book is full of useful information. Here are just some off the top of my head:

* The world is asynchronous so why model it in a synchronous way?
* When, and at what scale should you use actors
* How to implement D(distributed)DDD with akka.
* Different ways to handle state changes within an actor
* Handling long running operations within the actor
* Alternatives to using ask pattern, and when it is ok to use ask
* Where to keep message classes
* How to structure messages flow to achieve best throughput and latency
* How to prevent mailbox overflow
* Consistency vs. Scalability and how akka sharding can help with balancing them
* Isolating failures and self healing
* Preparing for failures even at the jvm level
* Maximize availability
* Find bottlenecks within jvm and akka itself


## For who?

I feel like when getting started with akka you are given this massive set of tools and you neither have an idea
which ones are best suited for certain situations, nor what are best practices.
There are gazillions of resources describing what is akka and how to get started with it. 
What is lacking though are set of best practices and common patterns.
Akka toolkit is really dangerous when put in the wrong hands. We, beginner/intermediate akka users, need those patterns and best practices 
compiled into one resource to protect against those mistakes. I think the book aims for this niche and nails it flawlessly.

## Final rant

One thing I missed was some kind of a bullet point list below each chapter with the most important statements. The book has so much material
that I had to write my own notes, otherwise I would not be able to retain all the information.






