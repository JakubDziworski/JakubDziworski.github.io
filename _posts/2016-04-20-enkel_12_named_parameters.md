---
layout: post
title: Creating JVM language [PART 12] - Named Function Arguments
categories: [Enkel]
tags: [enkel,jvm,asm,antlr,antlr4,antlr,java,language]
fullview: true
comments: true
---
## Sources

The project can be cloned from **[github repository](https://github.com/JakubDziworski/Enkel-JVM-language)**.  
The revision described in this post is **[62a99fe34f540f5cae7a48386b66d23e4b879046](https://github.com/JakubDziworski/Enkel-JVM-language/tree/62a99fe34f540f5cae7a48386b66d23e4b879046)**.

## Why do I need named arguments?

In java (like in most languages) the method call arguments are identified by indexes.
This seems reasonable for a methods with small amount of parameters and preferably different types.
Unfortunately there are many methods that neither have small amount of parameters nor different types.

If you've ever done some game programing you proably came across functions like this:

```java
Rect createRectangle(int x1,int y1,int x2, int y2) //createRectangle signature
``` 

I am more than sure you called it with wrong arguments order at least once.

Do you see the problem? The function has plenty parameters each the same type. It is very easy
to forget what is the order - the compiler doesn't care as long as types match.

Wouldn't it be awesome if you could explicitly specify a parameter without relying on the indexes?
That's where named arguments come in:

```java
createRectangle(25,50,-25,-50) //method invokation without named parameters :(
createRectangle(x1->25,x2->-25,y1->50,y2->-50) //method invokation with named parameters :)
```

The benefits from using named arguments are:

 * The order of arguments is unrestricted
 * The code is more readable
 * No need to jump between files to compare call with signature
 

## Grammar changes


```antlr
functionCall : functionName '('argument? (',' argument)* ')';
argument : expression              //unnamed argument
         | name '->' expression   ; //named argument
```

The function call can have one, or more (splitted by ',' character) arguments.
The rule ```argument``` comes in two flavours (unnamed and named).
Mixing named and unnamed arguments is not allowed.


## Reordering arguments

As described in [Creating JVM language [PART 7] - Methods](http://jakubdziworski.github.io/enkel/2016/04/05/enkel_7_methods.html)
, method parsing process is divided into two steps. First it finds all the signatures (declarations), and once it's done it starts parsing the bodies.
It is guaranteed that during parsing method bodies all the signatures are already available.

Using that characteristics the idea is to "transform" named call to unnamed call by getting parameters indexes from signature:

* Look for a parameter name in the signature that matches the argument name
* Get parameter index
* If the argument is at different index than a parameter reorder it.

![Reordering arguments](/assets/media/enkle_12/diagram.gif)

In the example above the x2 would be swapped with y1.

```java
public class ExpressionVisitor extends EnkelBaseVisitor<Expression> {
    //other stuff
    @Override
    public Expression visitFunctionCall(@NotNull EnkelParser.FunctionCallContext ctx) {
        String funName = ctx.functionName().getText();
        FunctionSignature signature = scope.getSignature(funName); 
        List<EnkelParser.ArgumentContext> argumentsCtx = ctx.argument();
        //Create comparator that compares arguments based on their index in signature
        Comparator<EnkelParser.ArgumentContext> argumentComparator = (arg1, arg2) -> {
            if(arg1.name() == null) return 0; //If the argument is not named skip
            String arg1Name = arg1.name().getText();
            String arg2Name = arg2.name().getText();
            return signature.getIndexOfParameter(arg1Name) - signature.getIndexOfParameter(arg2Name);
        };
        List<Expression> arguments = argumentsCtx.stream() //parsed arguments (wrong order)
                .sorted(argumentComparator) //Order using created comparator
                .map(argument -> argument.expression().accept(this)) //Map parsed arguments into expressions
                .collect(toList());
        return new FunctionCall(signature, arguments);
    }
}
```

That way the component responsible for generting bytecode does not distinct
named and unnamed arguments. It only sees FunctionCall as a collection of arguments (properly ordered) and a signature.
No modifications to bytecode generation are therefore needed.
        
## Example

The following Enkel class:

```java
NamedParamsTest {

    main(string[] args) {
        createRect(x1->25,x2->-25,y1->50,y2->-50)
    }

    createRect (int x1,int y1,int x2, int y2) {
        print "Created rect with x1=" + x1 + " y1=" + y1 + " x2=" + x2 + " y2=" + y2
    }
}
```
gets compiled into following bytecode:

```java
kuba@kuba-laptop:~/repos/Enkel-JVM-language$ javap -c NamedParamsTest.class 
public class NamedParamsTest {
  public static void main(java.lang.String[]);
    Code:
       0: bipush        25          //x1 (1 index in call)
       2: bipush        50          //y1 (3 index in call)
       4: bipush        -25         //x2 (2 index in call)
       6: bipush        -50         //y2 (4 index in call)
       8: invokestatic  #10                 // Method createRect:(IIII)V
      11: return

  public static void createRect(int, int, int, int);
    Code:
      //normal printing code 
}

```

As you can see the y1 and x2 arguments were swapped as expected.

The output is:

```
Created rect with x1=25 y1=50 x2=-25 y2=-50
```