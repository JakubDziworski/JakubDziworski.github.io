---
layout: post
title: Creating JVM language [PART 4] - Specifing Enkel language features
categories: [Enkel]
tags: [enkel,jvm,java,language]
fullview: true
comments: true
---
Since I already implemented first prototype version of Enkel compiler it is time
to take some time to actually design the features added in next iterations.  

There is a lot of verbosity in Java. It often protects you from mistakes, and minimizes unexpected "magic" implicit operations.
On the other hand it makes us write a lot of repeatable code.   

The main goal is to make Enkel opposite - as less verbose as possible.
This has downsides and upsisdes but is definately great for prototyping.
Here are some of the features I would like to implement in next iterations: 
 
| Feature                                                                                                             | Example                                                                                     |
|:--------------------------------------------------------------------------------------------------------------------|:--------------------------------------------------------------------------------------------|
| One file = one class = no need to specify 'class' keyword - just provide class name in the first line after imports | ```Car {}```                                                                                |
| Inheritance                                                                                                         | ```Car : Vehicle {}```                                                                      |
| Optional auto-generated getters,setters,builder,equals,hashcode                                                     | ```Car(getters,setters,hashequals,builder) : Vehicle  {}```                                 |
| Type inference                                                                                                      | ```var x = 5```                                                                             |
| Default parameters                                                                                                  | ```fun createPoint(Int x=0,Int y=0)```                                                      |
| Optional parameters names when calling method = more readable & any parameters order!                               | ```createPoint(5,0)```<br>  ``` createPoint(x->5,y->0)```<br>  ```createPoint(y->0,x->5)``` |
| Functions as objects                                                                                                | ```const f = (Int x=0, Int y=0) => x*y```                                                   |
| No static context                                                                                                   | ~~```static void x()```~~                                                  |
| == instead of equals by default.                                                                                    | object1 == object2                                                        |
