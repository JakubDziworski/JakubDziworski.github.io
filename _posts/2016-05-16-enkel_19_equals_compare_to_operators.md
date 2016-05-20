---
layout: post
title: Creating JVM language [PART 19] - Replacing equals and compareTo with operators
categories: [Enkel]
tags: [enkel,jvm,asm,antlr,antlr4,antlr,java,language,fields,compareTo,equals]
fullview: true
comments: true
---

<br>
<br>

![](/assets/media/enkel_19/equals.png)

<br>
<br>

## Sources

The project can be cloned from **[github repository](https://github.com/JakubDziworski/Enkel-JVM-language)**.  
The revision described in this post is **[7e6a08eaf4272cb07138fb1ef9d5c2bb7d300df8](https://github.com/JakubDziworski/Enkel-JVM-language/tree/7e6a08eaf4272cb07138fb1ef9d5c2bb7d300df8)**.


## Comparing objects in Java 

Comparing objects is one of the most surprising part of the language for Java's newcomers.
Diving into
the code directly without any theoretical background, one might find himself very confused by the results.
 
Moreover there are some traps that make the whole concept feel not deterministic. 
Let's take a look at this example:

```java
Integer a = 15;
Integer b = 15;
boolean areEqual = a == b;
```

There is an implicit boxing using ```Integer.valueOf(15)``` which returns **cached 
Integer object**. Because the reference is the same **areEqual is true**

After executing the code above, beginner Java programmer might think to himself
- "Great, I can compare objects with ```==```"

The next day he decides to change the values: 

```java
Integer a = 155;
Integer b = 155;
boolean areEqual = a == b;
```

and all of the sudden **areEqual is false** because 155 is above caching threshold.

Strings are also traps. If you create one by explicitly calling
new you get a new reference. On the other hand if you assign variable
to the string value ( " " notation) you get a pooled object.

The problem with this is that most of the times (I'd say 99%) we
are interested in comparing relation between objects, not the 
reference value. I really wish ```==``` meant relational equality,
and ```<``` , ```>``` , ```<=``` , ```>=``` called compareTo.

Instead of wishing let's just implement it then!

## Implementing  bytecode generation for ConditionalExpression

In [Creating JVM language [PART 10] - Conditional statements](http://jakubdziworski.github.io/enkel/2016/04/16/enkel_10_if_statement.html)
I introduced a way to compare primitive objects. The post describes
how the compare operators are created. The only thing that needs to be changed
is bytecode generation step.

Basically we need check if the value is primitive or reference.
If the object is reference then the equals or compareTo method calls are generated:

```java
public class ConditionalExpressionGenerator {
     
    //Constructor and fields

    public void generate(ConditionalExpression conditionalExpression) {
        Expression leftExpression = conditionalExpression.getLeftExpression();
        Expression rightExpression = conditionalExpression.getRightExpression();
        CompareSign compareSign = conditionalExpression.getCompareSign();
        if (conditionalExpression.isPrimitiveComparison()) {
            generatePrimitivesComparison(leftExpression, rightExpression, compareSign);
        } else {
            generateObjectsComparison(leftExpression, rightExpression, compareSign);
        }
        Label endLabel = new Label();
        Label trueLabel = new Label();
        methodVisitor.visitJumpInsn(compareSign.getOpcode(), trueLabel);
        methodVisitor.visitInsn(Opcodes.ICONST_0);
        methodVisitor.visitJumpInsn(Opcodes.GOTO, endLabel);
        methodVisitor.visitLabel(trueLabel);
        methodVisitor.visitInsn(Opcodes.ICONST_1);
        methodVisitor.visitLabel(endLabel);
    }

    private void generateObjectsComparison(Expression leftExpression, Expression rightExpression, CompareSign compareSign) {
        Parameter parameter = new Parameter("o", new ClassType("java.lang.Object"), Optional.empty()); // #1 
        List<Parameter> parameters = Collections.singletonList(parameter);
        Argument argument = new Argument(rightExpression, Optional.empty());
        List<Argument> arguments = Collections.singletonList(argument);
        switch (compareSign) { // #2
            case EQUAL:
            case NOT_EQUAL:
                FunctionSignature equalsSignature = new FunctionSignature("equals", parameters, BultInType.BOOLEAN); // #3
                FunctionCall equalsCall = new FunctionCall(equalsSignature, arguments, leftExpression);
                equalsCall.accept(expressionGenerator); // #4
                methodVisitor.visitInsn(Opcodes.ICONST_1); 
                methodVisitor.visitInsn(Opcodes.IXOR); // #5
                break;
            case LESS:
            case GREATER:
            case LESS_OR_EQUAL:
            case GRATER_OR_EQAL:
                FunctionSignature compareToSignature = new FunctionSignature("compareTo", parameters, BultInType.INT); // #6
                FunctionCall compareToCall = new FunctionCall(compareToSignature, arguments, leftExpression);
                compareToCall.accept(expressionGenerator);
                break;
        }
    }

    private void generatePrimitivesComparison(Expression leftExpression, Expression rightExpression, CompareSign compareSign) {
        leftExpression.accept(expressionGenerator);
        rightExpression.accept(expressionGenerator);
        methodVisitor.visitInsn(Opcodes.ISUB); 
    }
}
```


There are few sections worth explanation: 

**#1**

Equals method is declared in Object class as follows:

```java
public boolean equals(Object obj) {
        return (this == obj);
}
```

Therefore the parameter needs to be an ```java.lang.Object```. 
The name is irrelavant (```o``` seems fine).
There is no default value (```Optional.empty```)

**#2**

It's mandatory to distinguish whether the equality (```==``` or ```!=```),
or comparing (```>``` ```<``` ```>=``` or ```<=```) operators were used .
We could use ```compareTo``` for equality operator too but not all Classes 
implement ```Comparable``` interface.

**#3** 

As pointed out before equals method is named "equals" has one parameter
of type ```java.lang.Object``` and returns primitive ```boolean``` value. 

**#4**

Generate bytecode responsible for calling equals method. Take a look in
```CallExpressionGenerator``` class for more details on that.

**#5**

The equals returns true (1) if the objects
are equal or false (0) if the objects are different.
The primitives equality is calculated the other way around. The 
values are subtracted from each other. If the result is 0 it means
 values are equal, otherwise they are not.
To make things compatible, false needs to be swapped with true.
To do that I used XOR (Exclusive or) logical instruction.
The compareTo method on the other hand is very similar to primitive
comparison. It return 0 if equal too, so there is no need to make any changes.

**#6**

Creating call which represents ```compareTo``` call. ```compareTo```
was introduced before generics so it also takes ```java.lang.Object``` as a parameter,
but returns int.


## Example

The following Enkel class:

```java
EqualitySyntax {

 start {
    var a = new java.lang.Integer(455)
    var b = new java.lang.Integer(455)
    print a == b
    print a > b
 }
}
```

decompiled into Java looks like this:

```java
public class EqualitySyntax {
    public void start() {
        Integer var1 = new Integer(455);
        Integer var2 = new Integer(455);
        System.out.println(var1.equals(var2));
        System.out.println(var1.compareTo(var2) > 0);
    }

    public static void main(String[] var0) {
        (new EqualitySyntax()).start();
    }
}
```

As you can see ```==``` was sucesfully mapped to ```equals``` and ```>``` was mapped into ```compareTo```.

