---
layout: post
title: Creating JVM language [PART 11] - Default parameters
categories: [Enkel]
tags: [enkel,jvm,asm,antlr,antlr4,antlr,java,language]
fullview: true
comments: true
---
## Sources

The project can be cloned from **[github repository](https://github.com/JakubDziworski/Enkel-JVM-language)**.  
The revision described in this post is **[0d39a48855e15f5146bfa8ddee40effe84cf1093](https://github.com/JakubDziworski/Enkel-JVM-language/tree/0d39a48855e15f5146bfa8ddee40effe84cf1093)**.

## Java and default parameters

The absence of default parameters is one of the things I always hated in java.
Some people suggest using builder pattern but this solution leads to lots of boilerplate code.
I have no idea why java team have neglected this feature for so long. 
It actually turns out that it is not that hard to implement.

## Argument vs Parameter

Those terms are often mixed but actually they have different meaning. 
The simples way to put it is:

 * parameter - method signature
 * argument - method call

An argument is an expression passed when calling the method.
A parameter is a variable in method signature. 

## Concept

The idea is to look up method signature during call and get the parameter's default value from it.
That way no modifications regarding bytecode are required. The function call just "simulates"
a default value as if it was passed explicitly.


## Grammar changes

There is only one minor change to the ```functionParameterRule```:

```antlr
functionParameter : type ID functionParamdefaultValue? ;
functionParamdefaultValue : '=' expression ;
```

The function parameter consists of type, followed by name. Optionally ( '?' ) it is followed
by equals sign ('=') followed by some by default expression.

## Generating bytecode

Changes in this section are minor. New field (```defaulValue```) was introduced into ```FunctionParameter``` class.  
The field stores  ```Optional<Expression>``` object. If the parser founds defaultValue then the Optional consits of this value. 
Otherwise the Optional is empty.

```java
public class FunctionSignatureVisitor extends EnkelBaseVisitor<FunctionSignature> {

    @Override
    public FunctionSignature visitFunctionDeclaration(@NotNull EnkelParser.FunctionDeclarationContext ctx) {
       //other stuff
        for(int i=0;i<argsCtx.size();i++) { //for each parsed argument
        //other stuff
            Optional<Expression> defaultValue = getParameterDefaultValue(argCtx);
            FunctionParameter functionParameters = new FunctionParameter(name, type, defaultValue);
            parameters.add(functionParameters);
        }
        //other stuff
    }

    private Optional<Expression> getParameterDefaultValue(FunctionParameterContext argCtx) {
        if(argCtx.functionParamdefaultValue() != null) {
            EnkelParser.ExpressionContext defaultValueCtx = argCtx.functionParamdefaultValue().expression();
            return Optional.of(defaultValueCtx.accept(expressionVisitor));
        }
        return Optional.empty();
    }
}
```

Function Call bytecode generation visitor class has to additionaly do following steps:

* Check if there are not more arguments (method call) than parameters (method signature) 
* Get and evaluate default expressions for the arguments that are missing

The 'missing' arguments are defined as **arguments at index between last index in signature (exclusive) and
last index in function call (inclusive)**

Example:

signature:  fun(int x,int x2=5,int x3=4)

call: fun(2)

Missing arguments are x2(index 1) and x3(index 2) because last index in call is 0 and last index in signature is 2.

```java
public class ExpressionGenrator {
    public void generate(FunctionCall functionCall) {
        //other stuff
        if(arguments.size() > parameters.size()) {  
            throw new BadArgumentsToFunctionCallException(functionCall);
        }
        arguments.forEach(argument -> argument.accept(this));
        for(int i=arguments.size();i<parameters.size();i++) {
            Expression defaultParameter = parameters.get(i).getDefaultValue()
                    .orElseThrow(() -> new BadArgumentsToFunctionCallException(functionCall));
            defaultParameter.accept(this);
        }
        //other stuff   
    }
}
```
        
        
## Example

The following Enkel class:

```java
DefaultParamTest {

    main(string[] args) {
         greet("andrew")
         print ""
         greet("kuba","enkel")
    }

    greet (string name,string favouriteLanguage="java") {
        print "Hello my name is "
        print name
        print "and my favourite langugage is "
        print favouriteLanguage
    }
}
```
gets compiled into following bytecode:

```java
kuba@kuba-laptop:~/repos/Enkel-JVM-language$ javap -c DefaultParamTest
public class DefaultParamTest {
  public static void main(java.lang.String[]);
    Code:
       0: ldc           #8                  //push String "andrew" onto the stack
       2: ldc           #10   // push String "java" onto the stack  <-- implicit argument value
       4: invokestatic  #14                 // invoke static method greet:(Ljava/lang/String;Ljava/lang/String;)V
       7: getstatic     #20                 // get static field java/lang/System.out:Ljava/io/PrintStream;
      10: ldc           #22                 // push  empty String (empty line)
      12: invokevirtual #27                 // call Method "Ljava/io/PrintStream;".println:(Ljava/lang/String;)V to print empty line
      15: ldc           #29                 // push String "kuba"
      17: ldc           #31   // push String "enkel" <-- explicit argument value
      19: invokestatic  #14                 //invoke static method greet:(Ljava/lang/String;Ljava/lang/String;)V
      22: return

  public static void greet(java.lang.String, java.lang.String);
    Code:
       0: getstatic     #20                 // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #33                 // String Hello my name is
       5: invokevirtual #27                 // Method "Ljava/io/PrintStream;".println:(Ljava/lang/String;)V
       8: getstatic     #20                 // Field java/lang/System.out:Ljava/io/PrintStream;
      11: aload_0                           // load (push onto stack) variable at index 0 (first parameter of a method)
      12: invokevirtual #27                 // Method "Ljava/io/PrintStream;".println:(Ljava/lang/String;)V
      15: getstatic     #20                 // Field java/lang/System.out:Ljava/io/PrintStream;
      18: ldc           #35                 // String and my favourite langugage is
      20: invokevirtual #27                 // Method "Ljava/io/PrintStream;".println:(Ljava/lang/String;)V
      23: getstatic     #20                 // Field java/lang/System.out:Ljava/io/PrintStream;
      26: aload_1                           // load (push onto stack) variable at index 1 (second parameter of a method)
      27: invokevirtual #27                 // Method "Ljava/io/PrintStream;".println:(Ljava/lang/String;)V
      30: return
}
```

and the output is:

```
Hello my name is 
andrew
and my favourite langugage is 
java

Hello my name is 
kuba
and my favourite langugage is 
enkel
```