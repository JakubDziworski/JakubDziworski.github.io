---
layout: post
title: Spock testing framework cheatsheet
categories: [Java,Groovy,Spock]
tags: [spock,jvm,testing,groovy]
fullview: true
comments: true
---

If you are not familiar with spock it is testing framework for Groovy and Java.
This cheatsheet contains the most useful spock features regarding testing Java applications.
Most of this is copy-paste from official spock documentation.

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
    where: //optional:provide parametrized data (tables or socket) 
      
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
        7 | 4 | 4
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

We do not know which row failed

```groovy
maximum of two numbers   FAILED

Condition not satisfied:

Math.max(a, b) == c
    |    |  |  |  |
    |    7  0  |  7
    42         false
```

### Data Pipes

Left side must be Collection, String or Iterable.

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
@Issue("http://redmine/gdfgfdg)
@Subject
```


