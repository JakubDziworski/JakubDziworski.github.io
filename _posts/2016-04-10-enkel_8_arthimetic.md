---
layout: post
title: Creating JVM language [PART 8] - Arithmetic operations
categories: [Enkel]
tags: [enkel,jvm,asm,antlr,antlr4,antlr,java,language]
fullview: true
comments: true
---
## Sources

The project can be cloned from **[github repository](https://github.com/JakubDziworski/Enkel-JVM-language)**.  
The revision described in this post is **[1fc8131b2752e73776e91084ffeabbfa45fc6307](https://github.com/JakubDziworski/Enkel-JVM-language/tree/1fc8131b2752e73776e91084ffeabbfa45fc6307)**.

## Grammar changes
The basic arithmetic operations are:
 
 * Addition
 * Subtraction
 * Multiplication
 * Division

The only affected grammar component is "expression" rule.  
Expression is "something" that evaluates to a value (functon calls, values, variable references etc.).   
Statement does "something" but not necessarily evaluate to value (if blocks etc.).  
Since arithmetic operations return a value they are expressions:

```antlr
expression : varReference #VARREFERENCE
           | value        #VALUE
           | functionCall #FUNCALL
           |  '('expression '*' expression')' #MULTIPLY
           | expression '*' expression  #MULTIPLY
           | '(' expression '/' expression ')' #DIVIDE
           | expression '/' expression #DIVIDE
           | '(' expression '+' expression ')' #ADD
           | expression '+' expression #ADD
           | '(' expression '-' expression ')' #SUBSTRACT
           | expression '-' expression #SUBSTRACT
           ;
```

There are few things to clarify here.

The **```#```** notation means 'create callback for this rule alternative'.
Antlr would then generate methods like ```visitDIVIDE()```, ```visitADD()``` in ```EnkelVisitor``` interface.
It just a shortcut for creating new rules.

The rule's alternatives order is crucial here! Let's say we have following expression : ```1+2*3```.
There is ambiguity because it could be parse like this ```1+2=3  3*3=9```  , or ```2*3=6  6+1=7```.
Antlr resolves ambiguity by choosing the first alternative specified. Alternatives
order is therefore relative to an arithmetic operations order.

Expression in grouping parenthesis are intentionally put above the regular version in the rule.
This makes them higher priority. Thanks to that expressions like ```(1+2)*3``` are parsed in correct order.

## Mapping antlr context objects

Antlr generates new Classes and callbacks for each rule alternative (arithmetic expression).
It is good idea to however create custom classes for each operation. This will
make bytecode generation code way cleaner:

```java
public class ExpressionVisitor extends EnkelBaseVisitor<Expression> {

    //some other methods (visitFunctionCall, visitVaraibleReference etc)
    
    @Override
    public Expression visitADD(@NotNull EnkelParser.ADDContext ctx) {
        EnkelParser.ExpressionContext leftExpression = ctx.expression(0);
        EnkelParser.ExpressionContext rightExpression = ctx.expression(1);

        Expression leftExpress = leftExpression.accept(this);
        Expression rightExpress = rightExpression.accept(this);

        return new Addition(leftExpress, rightExpress);
    }

    @Override
    public Expression visitMULTIPLY(@NotNull EnkelParser.MULTIPLYContext ctx) {
        EnkelParser.ExpressionContext leftExpression = ctx.expression(0);
        EnkelParser.ExpressionContext rightExpression = ctx.expression(1);

        Expression leftExpress = leftExpression.accept(this);
        Expression rightExpress = rightExpression.accept(this);

        return new Multiplication(leftExpress, rightExpress);
    }
    
    //Division
    
    //Substration
}
```

Multiplcation,Addition,Division and Substraction  are just immutable
POJO objects, which store left and right expressions of the operation (1+2 - 1 is left,2 is right).

## Generating bytecode

Once the code is parsed and mapped into objects we can transform them
into bytecode.
To do that I created another Class (according to visitor pattern too) which
takes an object of type Expression and generates a bytecode. 

```java
public class ExpressionGenrator {

    //other methods (generateFunctionCall, generateVariableReference etc.)

    public void generate(Addition expression) {
        evaluateArthimeticComponents(expression);
        methodVisitor.visitInsn(Opcodes.IADD);
    }

    public void generate(Substraction expression) {
        evaluateArthimeticComponents(expression);
        methodVisitor.visitInsn(Opcodes.ISUB);
    }

    public void generate(Multiplication expression) {
        evaluateArthimeticComponents(expression);
        methodVisitor.visitInsn(Opcodes.IMUL);
    }

    public void generate(Division expression) {
        evaluateArthimeticComponents(expression);
        methodVisitor.visitInsn(Opcodes.IDIV);
    }
    
    private void evaluateArthimeticComponents(ArthimeticExpression expression) {
            Expression leftExpression = expression.getLeftExpression();
            Expression rightExpression = expression.getRightExpression();
            leftExpression.accept(this);
            rightExpression.accept(this);
    }
}
```

The arthimetic operations using bytecodes are very straightforward.
They take top two values from stack and  put a result back onto it.
No operands are required:

 * ```iadd``` - adds integers. Takes two values from the stack, adds them and pushes result back onto the stack
 * ```isub``` - substracts integers. Takes two values from stack, substracts them and pushes result back onto the stack
 * ```imul``` - multiplies integers. Takes two values from stack, multiplies them and pushes result back onto the stack
 * ```idiv``` - divides integers. Takes two values from stack, divides them and pushes result back onto the stack

The instructions for other types are corresponding.


## Result

The following Enkel code:

```
First {
        void main (string[] args) {
            var result = 2+3*4
        }
}
```
gets compiled into following bytecode:

```java
kuba@kuba-laptop:~/repos/Enkel-JVM-language$ javap -c First
public class First {
  public static void main(java.lang.String[]);
    Code:
       0: bipush        2 //push 2 onto the stack
       2: bipush        3 //push 3 onto the stack
       4: bipush        4 //push 4 onto the stack
       6: imul          //take two top values from the stack (3 and 4) and multiply them. Put result on stack
       7: iadd          //take two top values from stack (2 and 12-result of imul) and add em. Put result back on stack
       8: istore_1     //store top value from the stack into local variable at index 1 in local variable array of the curennt frame
       9: return
}
```

