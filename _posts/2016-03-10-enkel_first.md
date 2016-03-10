---
layout: post
title: Writing your own JVM language [PART 1] - Enkel
categories: [Enkel]
description: First post.
tags: [enkel,jvm,java]
fullview: true
comments: true
---

As I mentioned in previous post I am participating in ["Daj się poznać" contest](http://www.maciejaniserowicz.com/daj-sie-poznac). The goal of the contest is to implement any project and document it's development process by blogging.     
I came up with an idea of creating my own simple JVM language and compiler (because why the hell not)?     
The language is called **Enkel**, which means "simple" in swedish.   
The reasons why I decided to go with language running on JVM are:   

* JVM specification is very well documented.
* JVM languages can be mixed with each other - Enkel will be able to use Java libraries.
* I know Java fairly well (lexer,parser and compiler will be written in this language).
* There are some good Java libraries for manipulating bytecode.
* There are many things to "improve" in Java language.

Over the next 10 weeks I will be describing the implementation process, so stay tuned.
