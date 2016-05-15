---
layout: post
title: JUnit vs Spock + Spock Cheatsheet
categories: [Java,Groovy,Spock]
tags: [spock,jvm,testing,groovy]
fullview: true
comments: true
---

If you are not familiar with spock it is testing framework for Groovy and Java.
It's been stable for quite some time and I highly recommend you to check it out
if you are annoyed by Junit and Java's style of writing tests.


## What's wrong with JUnit + \<some mocking framework\> ?

The standard way of testing Java application is to use Junit
and some mocking framework (Mockito,EasyMock, PowerMock etc.).

Java combined with those frameworks makes it rather hard to 
write and read tests in medium and large sizes projects:
 
 * You cannot set title for a test (Junit5 introduces this feature but it is still in alpha). Instead you have
 to name your method in a ridiculous way like 'shouldAddToCartIfItemIsAvailaibleAndTheLimitIsNotExceededAnd.....'.
 * The intent of the test is blurred by all those Java and mocking framework verbosity like
   Collections.singletonList()'s,replay's,verify's or any(MyAwesomeAbstractFactoryBaseClass.class)'s and many more.
 * There is no good way to separate sections responsible for different phases (given,when,then). 
   Some people use comments to mark those sections but I think it's even worse than not having them at all.
 * Java is certainly not easy language for building "expected" objects - everything is so verbose.
   Once again - it hides the intent of a test.
 * Parametrizied tests are kinda weird too. They must be stored in fields,
   and you can only have one set of them per test class.
 * Since parametrized test are not simple you usually  
  write gazillion of separate methods - each covering different case, or even worse
  skip some cases hoping noone will notice :).

If tests are hard to write we usually think of them
as something painful and start to neglect them.
Avoiding or delaying writing tests leads to the situation where application cannot be trusted anymore. 
We then become afraid of making any changes because other part of the app might 
break in some bizarre way.

It shouldn't be this way. Test should be easy and fun to write. After all
they are like a cherry on top, proving that the features are implemented correctly.

In my opinion the most important responsibility of the test is to be as most readable as possible.
Business changes to the project are introduced all the time. If we change something in the 
application we have to change test too (unless you're applying open-closed principle, which I've never heard
of anyone successfully adapting :D). If tests are hard to read there is a big problem.

On the other hand - it's just my opinion, who am I to judge? Do you feel similar about this topic
or is it just me? If you disagree, or have some objections leave a comment!

# Spock 

Spock is both testing and mocking framework. What's more it extends
Junit runner so it can be runned by the tools you
used for your tests before.

The best thing about Spock is that **it's basically a DSL (domain specifing language) for writing tests**.
It's based on Groovy and is designed particularly testing. It introduces some syntax 
features just for that purpose. You may therefore expect some neat stuff in it (which is indeed correct).

Groovy is kinda like a scripting version of Java - simple, less verbose but retains all the power of JVM.


Benefits from using spock over Junit + mocking framework:

* Groovy - less verbose than Java
* Additional syntax features designed for testing
* Integrated stubbing and mocking
* Extends Junit runner
* Easy parametrized testing
* Labels for all phases of a test (given,when,then...)
* Ability to document methods easily (unlike weird Junit method's name pattern)
* Many more specified below


## Cheatsheet

This cheatsheet contains the most useful spock features regarding testing Java applications.
Most of this is copy-paste from official spock documentation. I compiled it 
while I was learning the framework to have all information in one place. 
I figure out since it's already compiled why not share it on a blog too.


# Basics

### Specification template

```groovy
class MyFirstSpecification extends Specification {
  // fields
  // fixture methods
  // feature methods
  // helper methods
}
```

### Fixture Methods

```groovy
def setup() {}          // run before every feature method
def cleanup() {}        // run after every feature method
def setupSpec() {}     // run before the first feature method
def cleanupSpec() {}   // run after the last feature method
```

### Blocks order

```groovy
    given: //data initialization goes here (includes creating mocks)
    when: //invoke your test subject here and assign it to a variable
    then: //assert data here
    cleanup: //optional
    where: //optional:provide parametrized data (tables or pipes) 
      
```
or 

```groovy    
    given:
    expect: //combines when with then
    cleanup: 
    where:
```        

![Blocks](/assets/media/spock-cheatsheet/Blocks2Phases.png)

### Junit comparison 

|Spock |	JUnit|
|-----|------|
| Specification | Test class |
| setup() | @Before |
| cleanup()  | @After |
| setupSpec()  | @BeforeClass |
| cleanupSpec()  | @AfterClass |
| Feature  | Test |
| Feature method  | Test method |
| Data-driven feature  | Theory |
| Condition | Assertion |
| Exception condition | @Test(expected=…​) |
| Interaction | Mock expectation (e.g. in Mockito) |

# Data Driven Testing

### Data Tables

```groovy
class Math extends Specification {
    def "maximum of two numbers"(int a, int b, int c) {
        expect:
        Math.max(a, b) == c

        where:
        a | b | c
        1 | 3 | 3
        7 | 4 | 7
        0 | 0 | 0
    }
}
```

Input data can also be seperated with expected parameters using ```||```:

```groovy
    where:
    a | b || c
    3 | 5 || 5
    7 | 0 || 7
    0 | 0 || 0
```

### Unrolling

A method annotated with @Unroll will have its rows from data table reported independently:

```groovy
@Unroll
def "maximum of two numbers"() { ... }
```

#### With unroll

```groovy
maximum of two numbers[0]   PASSED
maximum of two numbers[1]   FAILED

Math.max(a, b) == c
    |    |  |  |  |
    |    7  0  |  7
    42         false

```

#### Without unroll

We have to figure out which row failed manually

```groovy
maximum of two numbers   FAILED

Condition not satisfied:

Math.max(a, b) == c
    |    |  |  |  |
    |    7  0  |  7
    42         false
```

### Data Pipes

Right side must be Collection, String or Iterable.

```groovy
where:
a << [3, 7, 0]
b << [5, 0, 0]
c << [5, 7, 0]
```

### Multi-Variable Data Pipes

```groovy
where:
[a, b, c] << sql.rows("select a, b, c from maxdata")
```

```groovy
where:
row << sql.rows("select * from maxdata")
// pick apart columns
a = row.a
b = row.b
c = row.c
```

#### Ignore some variable

```groovy
where:
[a,b] << [[1,2,3],[1,2,3],[4,5,6]]
[a, b, _, c] << sql.rows("select * from maxdata")
```

### Combine data tables,pipes and assignments

```groovy
where:
a | _
3 | _
7 | _
0 | _

b << [5, 0, 0]

c = a > b ? a : b
```

### Unrolled method names parameters

```groovy
def "#person is #person.age years old"() { ... } // property access
def "#person.name.toUpperCase()"() { ... } // zero-arg method call
```





# Interaction Based Testing

## Mocking 

### Create mock

Mocks are Lenient (return default value for undefined mock calls)

```groovy
Subscriber subscriber = Mock()
def subscriber2 = Mock(Subscriber)
```

### Using mock

```groovy
def "should send messages to all subscribers"() {
    when:
    publisher.send("hello")

    then:
    1 * subscriber.receive("hello") //subsriber should call receive with "hello" once.
    1 * subscriber2.receive("hello")
}
```

### Cardinality

```groovy
1 * subscriber.receive("hello")      // exactly one call
0 * subscriber.receive("hello")      // zero calls
(1..3) * subscriber.receive("hello") // between one and three calls (inclusive)
(1.._) * subscriber.receive("hello") // at least one call
(_..3) * subscriber.receive("hello") // at most three calls
_ * subscriber.receive("hello")      // any number of calls, including zero
                                     // (rarely needed; see 'Strict Mocking')
```

### Constraints

#### Target

```groovy
1 * subscriber.receive("hello") // a call to 'subscriber'
1 * _.receive("hello")          // a call to any mock object
```

#### Method

```groovy
1 * subscriber.receive("hello") // a method named 'receive'
1 * subscriber./r.*e/("hello")  // a method whose name matches the given regular expression
                                // (here: method name starts with 'r' and ends in 'e')
```
                                
#### Argument

```groovy                                
1 * subscriber.receive("hello")     // an argument that is equal to the String "hello"
1 * subscriber.receive(!"hello")    // an argument that is unequal to the String "hello"
1 * subscriber.receive()            // the empty argument list (would never match in our example)
1 * subscriber.receive(_)           // any single argument (including null)
1 * subscriber.receive(*_)          // any argument list (including the empty argument list)
1 * subscriber.receive(!null)       // any non-null argument
1 * subscriber.receive(_ as String) // any non-null argument that is-a String
1 * subscriber.receive({ it.size() > 3 }) // an argument that satisfies the given predicate
                                          // (here: message length is greater than 3)                                
```

### Specify mock calls at creation 

```groovy
class MySpec extends Specification {
    Subscriber subscriber = Mock {
        1 * receive("hello")
        1 * receive("goodbye")
    }
}
```

### Group interactions

```groovy
with(mock) {
    1 * receive("hello")
    1 * receive("goodbye")
}
```

## Stubbing

Stubs do not have cardinality (matches invokation anyTimes)

```groovy
def subsriber = Stub(Subscriber)
...
subscriber.receive(_) >> "ok"
```

Whenever the subscriber receives a message, make it respond with 'ok'


### Returning different values on sucessive calls 

```groovy
subscriber.receive(_) >>> ["ok", "error", "error", "ok"]
subscriber.receive(_) >>> ["ok", "fail", "ok"] >> { throw new InternalError() } >> "ok"
```

# Extensions

```groovy
@Ignore(reason = "TODO")
@IgnoreRest
@IgnoreIf({ spock.util.environment.Jvm.isJava5()) })
@Requires({ os.windows })
@Timeout(5)
@Timeout(value = 100, unit = TimeUnit.MILLISECONDS)
@Title("This tests if..."
@Narrative("some detailed explanation")
@Issue("http://redmine/23432")
@Subject
```


