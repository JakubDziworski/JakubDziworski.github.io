---
layout: post
title: Creating JVM language [PART 14] - Handling other primitive types
categories: [Enkel]
tags: [enkel,jvm,asm,antlr,antlr4,antlr,java,language]
fullview: true
comments: true
---
## Sources

The project can be cloned from **[github repository](https://github.com/JakubDziworski/Enkel-JVM-language)**.  
The revision described in this post is **[30e678fea0847b84bb21154648104d343540908f](https://github.com/JakubDziworski/Enkel-JVM-language/tree/30e678fea0847b84bb21154648104d343540908f)**.

## New supported types

So far Enkel supported integers and Strings only.
It is time to include all the other primitive types.
This step was also necessary to prepare a compiler 
for upcoming object oriented features coming soon (so far creating objects
other than String is not possible).

## Many versions of the same instruction - let's generalize this.

There are many bytecode instructions that only differ from each other by type.
Let's take a look at the return instruction as an example:

 * return - returns from a method
 * ireturn - returns integer value (pops it off the stack) from a method
 * freturn - returns float value
 * dreturn - returns double value
 * lreturn - returns long value
 * areturn - returns reference value

It might be tempting to just add cases for each type in each section that emits bytecode instruction.
It would however result in awful ifology - we don't want that.
Instead I decided to make an ```TypeSpecificOpcodes``` enum that stores all the opcodes for each type respectively and is reachable by ```Type``` enum:

```java
public enum TypeSpecificOpcodes { 

    INT (ILOAD, ISTORE, IRETURN,IADD,ISUB,IMUL,IDIV), //values (-127,127) - one byte.
    LONG (LLOAD, LSTORE, LRETURN,LADD,LSUB,LMUL,LDIV),
    FLOAT (FLOAD, FSTORE, FRETURN,FADD,FSUB,FMUL,FDIV),
    DOUBLE (DLOAD, DSTORE, DRETURN,DADD,DSUB,DMUL,DDIV),
    VOID(ALOAD, ASTORE, RETURN,0,0,0,0),
    OBJECT (ALOAD,ASTORE,ARETURN,0,0,0,0);

    TypeSpecificOpcodes(int load, int store, int ret, int add, int sub, int mul, int div) {
        //assign each parameter to the field
    }
    
    //getters
```

The (type aware) instructions used so far are:

* load - load variable 
* store - store variable 
* ret - return
* add - add two last values from the stack 
* sub - substract two last values from the stack 
* mul - multiply two last values from the stack 
* div - divide two last values from the stack

The TypeSpecificOpcodes is composited in ```BultInType``` enum:

```java
public enum BultInType implements Type {
    BOOLEAN("bool",boolean.class,"Z", TypeSpecificOpcodes.INT),
    
    //other members
    
    BultInType(String name, Class<?> typeClass, String descriptor, TypeSpecificOpcodes opcodes) {
        //assign to fields
    }
    
    @Override
    public int getMultiplyOpcode() {
        return opcodes.getMultiply();
    }
```
No whenever multiply two values is taking place, there is no need to find opcode specific for expression
 type - it is already known by a Type. Just simply:

```java
public void generate(Multiplication expression) {
    evaluateArthimeticComponents(expression);
    Type type = expression.getType();
    methodVisitor.visitInsn(type.getMultiplyOpcode());
}
```

## Example

The following Enkel class:

```java
main(string[] args) {
        var stringVar = "str"
        var booleanVar = true
        var integerVar = 2745 + 33
        var doubleVar = 2343.05
        var sumOfDoubleVars =  23.0 + doubleVar
    }
```

is compiled into following bytecode:

```java
kuba@kuba-laptop:~/repos/Enkel-JVM-language$ javap -c AllPrimitiveTypes.class 
public class AllPrimitiveTypes {
  public static void main(java.lang.String[]);
    Code:
       0: ldc           #8                  // String str
       2: astore_1                          //store it variable
       3: ldc           #9                  // int 1 - bool values are represented as ints in JVM
       5: istore_2                          //store as int 
       6: ldc           #10                 // int 2745 
       8: ldc           #11                 // int 33
      10: iadd                              // iadd - add integers
      11: istore_3                          //store result in integer varaible
      12: ldc           #12                 // float 2343.05f 
      14: fstore        4                   //store in float variable
      16: ldc           #13                 // float 23.0f 
      18: fload         4                   //load integer varaible (from index 4)
      20: fadd                              //add float variables
      21: fstore        5                   //store float result
      23: return
}

```

As you can see the opcodes for the instructions are of the types corresponding
 to the expected types of a statements / expressions.