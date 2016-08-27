---
layout: post
title: Github Code Search - Programmers' Goldmine
categories: [Tools]
tags: [git,github,search,code,github search,advanced,java,scala,akka,spring]
fullview: true
comments: true
---

Learning new language or framework can sometimes be a struggle. Traditional approach is to read documentation which explains the concepts, and provides simple examples.
Sometimes that might be enough, but what those documentations are often lacking are some advanced examples and usages in real projects.

Coming across a problem which is not described in documentation, most people look for solution on stackoverflow (or dig through sources).
However the framework you are using might not be in the game for long enough to fill stackoverflow with every question you come up with.
 
Have you ever been stuck with a problem and thought to yourself:

_**"I know someone must have solved this before! Why there is no stackoverflow answer to this problem?"**_

You are right - someone has probably already solved it. And it's very likely the solution has been pushed to github.
It's just a matter of finding it. Programmers are more likely to solve the issues themselves rather than ask random people on the internet about it.

# Github search code

[Github search](https://github.com/search) provides a way to query repos in a various ways. One of them is [searching code](https://developer.github.com/v3/search/#search-code).
This is extremly powerful feature. Every line ever written by anybody can be found with simple queries.
The "good" thing about github is that the private repos are not free, so there are many projects implicitly shared to public by people who just want to backup their code. This is a goldmine of information!


## Examples

Below are some of the examples which I find github search code is handy for.

### Learning new api

Have you ever been stuck with 3rd party api, and unable to find similar code snippets for your case?

I was recently in need to use **akka streams** to read a huge file and pass the results to another file instantly.
The [documentation](http://doc.akka.io/docs/akka/2.4/scala/stream/stream-io.html#Streaming_File_IO) regarding
 this topic is good but short and could provide more examples.

Github advanced search to the rescue. After few clicks I found an awesome piece of code
that streams the csv file modifies it and dumps to another file!

![filepaths_example](/assets/media/github_search/filepaths_example.gif)


### Finding projects using technologies you are interested in

Let's say you want to learn **Spring MVC**, **Hibernate** and testing with **Spock**. You could go to the docs
of each libraries, and learn them one by one... or just find a project which integrates all of them.

Most platforms have some kind of dependency management tools. In case of **Java** that is 
usually **Maven** which stores all dependencies information in `pom.xml` file.

You can therefore query keywords and filename to find the projects you are interested in:

```
spring hibernate spock filename:pom.xml
```
This method is also
great if you are looking for projects to contribute to.

![find_technology](/assets/media/github_search/find_technology.gif)

### Integrating with external services

Looking for a quick way to integrate with github api using your favourite language? No problem - just look for
the repos with the api url and filter by language:

```
api.github.com language:scala
```

![find_integrations](/assets/media/github_search/find_integrations.gif)

### Configuration

It also wouldn't hurt to take a look at configuration files of real big projects.
This might be extremly helpful, particularly in case of immature frameworks.

Let's take a look how to configure akka cluster. Such configuration should
contain `ClusterActorRefProvider` keyword and reside in file with `.conf` extension (usually `application.conf`):

```
ClusterActorRefProvider extension:conf
```

![find_configuration](/assets/media/github_search/find_configuration.gif)

# Conclusion

Github search is underrated yet extremly powerful tool for learning new apis,solving issues and finding repos you might be interested in.
This is a great way to quickly get started with new framework - finding code snippets that are similar to what you want to achieve has never been easier.
It also makes you feel less alone with the issues you encounter - it's likely some has already solved it. Likewise, discovering interesting projects with this
search engine is just a matter of minutes.