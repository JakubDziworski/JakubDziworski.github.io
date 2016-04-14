---
layout: post
title: Creating JVM language [PART 9] - Return
categories: [Enkel]
tags: [enkel,jvm,asm,antlr,antlr4,antlr,java,language]
fullview: true
comments: true
---
## Sources

The project can be cloned from **[github repository](https://github.com/JakubDziworski/Enkel-JVM-language)**.  
The revision described in this post is **[83102b4c3f979c8d3e82abe35c91d3b14d37f1ab](https://github.com/JakubDziworski/Enkel-JVM-language/tree/83102b4c3f979c8d3e82abe35c91d3b14d37f1ab)**.

## Grammar changes

I defined a new rule called "**returnStatement**". 

You may wonder why is it not "returnExpression"?
After all an expression is something that evaluates to a value (as described in [the previous post](http://jakubdziworski.github.io/enkel/2016/04/10/enkel_8_arthimetic.html) ) . Doesn't a return statement evaluate to a value?

This may seem confusing but it turns out that return does not evaulate to a value.
In Java the following code would not make sens:
``` int x = return 5; ``` , and same thing is with enkel.
In other words expression is essentially something that can be assigned to a variable.
That is why the return is a statement not an expression:

```antlr
statement : variableDeclaration
           //other statements rules
           | returnStatement ;

variableDeclaration : VARIABLE name EQUALS expression;
printStatement : PRINT expression ;
returnStatement : 'return' #RETURNVOID
                | ('return')? expression #RETURNWITHVALUE;
```

The return statement comes in two versions:

* RETURNVOID - used in void methods. Does not take an expression. Requires 'return' keyword
* RETURNWITHVALUE - used in non-void methods. Does require an expression. Does not require 'return' keyword (optional). 

It is possible to return value explicitly and implicitly:

```java
SomeClass {
    fun1 {
       return  //explicitly return from void method
    }
    
    fun2 {
        //implicitly return from void method
    }
    
    int fun2 {
        return 1  //explicitly return "1" from int method
    }
    
    int fun3 {
        1  //implicitly return "1" from int method
    }
}
```

The above code results in following parse tree:

![Parse Tree](/assets/media/enkel_9/parse_tree.png)

You may notice that the parser did not resolve implicit return statement in fun2.
This is due to the fact that the block is empty and matching "empty" as return statement is not a good idea.
This missing return statements are added at bytecode generation phase.

## Mapping antlr context objects

Parsed return statements converted from antlr context classes into POJO ```ReturnStatement``` objects.
The purpose of this step is to feed compiler only with data required for bytecode generation. 
Getting data from antlr generated objects to generate bytecode would result in ugly unreadable code.

```java
public class StatementVisitor extends EnkelBaseVisitor<Statement> {

    //other stuff
    
    @Override
    public Statement visitRETURNVOID(@NotNull EnkelParser.RETURNVOIDContext ctx) {
        return new ReturnStatement(new EmptyExpression(BultInType.VOID));
    }
    
    @Override
    public Statement visitRETURNWITHVALUE(@NotNull EnkelParser.RETURNWITHVALUEContext ctx) {
        Expression expression = ctx.expression().accept(expressionVisitor); 
        return new ReturnStatement(expression);
    }   
}

```


## Detecting implicit void return

If there is a implicit return from a void method, no return statement is detected during parse time.
That is why it is necessary to detect this scenario and append return statement at generation time.

```java
public class MethodGenerator {
    //other stuff
    private void appendReturnIfNotExists(Function function, Block block,StatementGenerator statementScopeGenrator) {
        Statement lastStatement = block.getStatements().get(block.getStatements().size() - 1);
        boolean isLastStatementReturn = lastStatement instanceof ReturnStatement;
        if(!isLastStatementReturn) {
            EmptyExpression emptyExpression = new EmptyExpression(function.getReturnType());
            ReturnStatement returnStatement = new ReturnStatement(emptyExpression);
            returnStatement.accept(statementScopeGenrator);
        }
    }
}
```
This method detects if the last statement in the method is a ReturnStatement.
If not it generates return instruction.

## Generating bytecode

```java
public class StatementGenerator {
    //oher stuff
    public void generate(ReturnStatement returnStatement) {
        Expression expression = returnStatement.getExpression();
        Type type = expression.getType();
        expression.accept(expressionGenrator); //generate bytecode for expression itself (puts the value of expression onto the stack)
        if(type == BultInType.VOID) {
            methodVisitor.visitInsn(Opcodes.RETURN);
        } else if (type == BultInType.INT) {
            methodVisitor.visitInsn(Opcodes.IRETURN);
        }
    }
}
```

As an example the statements ```return 5``` would result in following steps:

* get expression from returnStatement ("5" - which is of type "Value" - deducted during parsing).
* generate bytecode for "5" expression (```expression.accept(expressionGenerator)``` calls ```ExpressionGenerator.generate(Value value)``` - visitor pattern).
* Bytecode generation results in a new value (5) pushed onto operand stack
* IRETURN instruction invocation takes a value from operand stack and returns it. 

Which generates bytecode:

``` 
 bipush        5
 ireturn
```

## Example

The following Enkel class:

```java
SumCalculator {

    main(stirng[] args) {
        print sum(5,2)
    }

    int sum (int x ,int y) {
        x+y
    }
}
```
gets compiled into following bytecode:

```java
kuba@kuba-laptop:~/repos/Enkel-JVM-language$ javap -c  SumCalculator
public class SumCalculator {
  public static void main(java.lang.String[]);
    Code:
       0: getstatic     #12                 //get static field java/lang/System.out:Ljava/io/PrintStream;
       3: bipush        5
       5: bipush        2
       7: invokestatic  #16                 // call method sum (with the values on operand stack 5,2)
      10: invokevirtual #21                 // call method println (with the value on stack - the result of method sum)
      13: return                           //return

  public static int sum(int, int);
    Code:
       0: iload_0
       1: iload_1
       2: iadd
       3: ireturn //return the value from operand stack (result of iadd)
}

```

