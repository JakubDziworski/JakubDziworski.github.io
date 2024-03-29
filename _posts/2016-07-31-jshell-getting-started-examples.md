---
layout: post
title: JShell - Java 9 interpreter (REPL) - Getting Started and Examples
categories: [Java]
tags: [jshell,java 9,9,java,api,repl,interpreter,getting,started,examples]
fullview: true
comments: true
---


Many compiled languages include tools (sometimes called REPL) for statements interpretation.
Using these tools you can test code snippets rapidly without creating project.

Take Scala as an example. Compilation can sometimes take a long time, but using repl each statement is executed instantly! That's
great when you are getting started with the language. Each expression gives you returned value and it's type - that's very valuable
information.

In java, instead, we have to create a test or main method which prints results and needs to be recompiled every time you make a change.

# When?
 
JShell will be introduced in [Java 9 realease](http://www.java9countdown.xyz/). You can however get early access build 
on [https://jdk9.java.net/](https://jdk9.java.net/).

# Running

Once you downloaded jdk9 there is a jshell executable in a bin directory. I suggest running it in verbose (`-v`) mode for the first time:

```bash
kuba@kuba-laptop:~/repos$ jdk-9/bin/jshell -v
|  Welcome to JShell -- Version 9-ea
|  For an introduction type: /help intro


jshell> 
```

You can go back to non verbose mode using `/set feedback normal`.

## Default imports

By default you get a set of common imports:

```bash
jshell> /imports
|    import java.util.*
|    import java.io.*
|    import java.math.*
|    import java.net.*
|    import java.util.concurrent.*
|    import java.util.prefs.*
|    import java.util.regex.*
```

You can add your own any time.

## Expressions

You can type any valid java expression, and it will tell you the returned **value**, it's **type** and **assign** it to a **variable**:

```bash
jshell> 3+3
$1 ==> 6
|  created scratch variable $9 : int

jshell> $1
$1 ==> 6
|  value of $1 : int

```

## Variables

It is possible to declare variables and name them. Once you do that they become visible in the scope.

```bash
jshell> int x=5
x ==> 5
|  created variable x : int

jshell> x
x ==> 5
|  value of x : int
```

## Methods

You can also define methods and even replace them:

```bash
jshell> void helloJShell() { System.out.println("hello JShell"); }
|  created method helloJShell()

jshell> helloJShell();
hello JShell

jshell> void helloJShell() { System.out.println("wow, I replaced a  method"); }
|  modified method helloJShell()
|    update overwrote method helloJShell()

jshell> helloJShell()
wow, I replaced a  method

```

## Commands

Aparat from language syntax you can execute jshell commands. Some of the most useful ones (`/help` to list all of them) are:

### listing variables

```bash
jshell> /vars
|    int x = 0
|    double j = 0.5
```

### listing methods:

```bash
jshell> /methods
|    printf (String,Object...)void
|    helloJShell ()void
```
The printf method is defined by default.

### listing sources

```bash
jshell> /list
  14 : helloJShell();
  15 : void helloJShell() { System.out.println("wow, I replaced a  method"); }
  16 : helloJShell()
```

### editing sources in external editor

```bash
jshell> /edit helloJShell
```
Opens external editor, and replaces helloJShell method.


# Example use cases

After 20 years of Java without REPL one might wonder what scenarios are suitable for JShell.
Here are some examples.

## Veryfing return type
Remember the time you learned that dividing two integers in Java does not result in floating number? For some time
I was convinced that both numerator and denominator have to be floating for a result to be floating too. Let's test that:

```bash
jshell> 1/2
$1 ==> 0
|  created scratch variable $1 : int

jshell> 1.0/2
$2 ==> 0.5
|  created scratch variable $2 : double

jshell> 1/2.0
$3 ==> 0.5
|  created scratch variable $3 : double

jshell> 1.0f/2
$4 ==> 0.5
|  created scratch variable $4 : float

jshell> 1/2.0f
$5 ==> 0.5
|  created scratch variable $5 : float
```

Turns out only one of them has to be floating.

## Testing Java niuanses

Did you know that comparing autoboxed integers references which values are from range -128 to 127 (inclusive) returns true (they are cached)?
You can verify that with shell in a matter of seconds:

```bash
jshell> Integer i1 = 127
i1 ==> 127

jshell> Integer i2 = 127
i2 ==> 127

jshell> i1 == i2
$35 ==> true

jshell> Integer i2 = 128
i2 ==> 128

jshell> Integer i1 = 128
i1 ==> 128

jshell> i1 == i2
$38 ==> false
```

## Formatting

Sometimes the logs need to be verbose and properly formatted. This is tedious task and usually leads to few recompile cycles which
significantly slows us down. Imagine you forgot what was the format sign responsible for integers. You can quickly verify that:

Let's try `%i` (integer):

```bash
jshell> printf("I got %i apple",1)
|  java.util.UnknownFormatConversionException thrown: Conversion = 'i'
|        at Formatter$FormatSpecifier.conversion (Formatter.java:2691)
|        at Formatter$FormatSpecifier.<init> (Formatter.java:2717)
|        at Formatter.parse (Formatter.java:2565)
|        at Formatter.format (Formatter.java:2507)
|        at PrintStream.format (PrintStream.java:977)
|        at PrintStream.printf (PrintStream.java:873)
|        at printf (#s8:1)
|        at (#51:1)
```

Oops, maybe `%d` (decimal) :

```shell
jshell> printf("I got %d apple",1)
I got 1 apple
```


# Conclusion

JShell is a very useful tool for prototyping and testing Java code snippets. Even though it is not yet officially released I highly recommend checking it out.
There is also a JShell Java api which allows you to evaluate JShell from java.
Once the java 9 is out I bet there will be JShell integrations in most popualar IDEs - this will make
using it even more handy.