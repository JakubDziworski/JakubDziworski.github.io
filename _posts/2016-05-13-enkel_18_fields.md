---
layout: post
title: Creating JVM language [PART 18] - Fields
categories: [Enkel]
tags: [enkel,jvm,asm,antlr,antlr4,antlr,java,language,fields]
fullview: true
comments: true
---
## Sources

The project can be cloned from **[github repository](https://github.com/JakubDziworski/Enkel-JVM-language)**.  
The revision described in this post is **[550449af09030ced25653dfc0961b2cbfd05bbcb](https://github.com/JakubDziworski/Enkel-JVM-language/tree/550449af09030ced25653dfc0961b2cbfd05bbcb)**.


## Syntax 

The syntax is simplified version of Java's. You can specify type and name of a field.
There are no are no modifiers and keywords like 'static','volatile','transient' etc. I am trying to keep it simple so far.

```java
Fields {

    int field

    start {
        field = 5
        print field
    }
}
```

## Grammar changes

Until now, you could only define methods in class body scope.
It's time to introduce fields:

```java
classBody :  field* function* ;
field : type name;
```

I also added assign statement for already defined variables:

```java
assignment : name EQUALS expression;
```

### Why I have not implemented assign statement so far?

To make use of fields you have to assign them to something. Turns out
I have not yet implemented such a basic thing as assignment statement for already declared variables.
Why didn't I do that? Well It was kind of on purpose.

The reason behind it is I would like the variables to be constant (immutable).
Assigning means changing state - changing state lead to many issues (synchronization,side effects,memory leaks).

Have you ever read a Java code and that looks something like this:

```java
Stuff trustMeIWontModifyYourArg(SomeObject arg) {
    ... 999 lines of code 
    arg = null; //or some other nasty hidden stuff
    ...another 999 lines of code
}
```

By reading the signature you probably thought to yourself - "hmmm... does this method modify argument? Well, it does not 
have a final modifier but most of us (Java programmers) neglect it. Judging by it's name it should not modify my args so let's just use it."

Two hours later you randomly get **NullPointerException somewhere else** in your code. **The method modified
your argument.**

If you have no side effects in your methods you can easily make them parallel
without worrying about synchronization issues and other nasty stuff. Such methods does
not have a state = therea are no side effects! The easiest way to achieve 
no side effects is to use values (constant variables) only.

You can learn more about statements and what's wrong with them
in **[awesome talk by Uncle Bob (the talk about assignments starts at 11:15): 
https://youtu.be/7Zlp9rKHGD4?t=11m15s](https://youtu.be/7Zlp9rKHGD4?t=11m15s)**. Check it out!

## Generating bytecode

### Declaring field

To declare a field you use asm's library visitField method. It adds 
the field to the ```fields[]``` member in the [class structure](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.1) and automatically
increases the ```fields_count``` counter:

```java
public class FieldGenerator {

    private final ClassWriter classWriter;

    public FieldGenerator(ClassWriter classWriter) {
        this.classWriter = classWriter;
    }

    public void generate(Field field) {
        String name = field.getName();
        String descriptor = field.getType().getDescriptor();
        FieldVisitor fieldVisitor = classWriter.visitField(Opcodes.ACC_PUBLIC, name,descriptor, null, null);
        fieldVisitor.visitEnd();
    }
}
```

### Reading field

To get a field you need:

 * field name
 * field type descriptor (if the field is of type int it'd be mean "I")
 * owner internal name (if the field is owned by com.yourcompany.Car then it'd be "com/yourcompany/Car")

```java
public class ReferenceExpressionGenerator {

     //constructor and fields

    public void generate(FieldReference fieldReference) {
        String varName = fieldReference.geName();
        Type type = fieldReference.getType();
        String ownerInternalName = fieldReference.getOwnerInternalName();
        String descriptor = type.getDescriptor();
        methodVisitor.visitVarInsn(Opcodes.ALOAD,0);
        methodVisitor.visitFieldInsn(Opcodes.GETFIELD, ownerInternalName,varName,descriptor);
    }
}
```

 * ```ALOAD,0``` - gets "this" object which is local variable at index 0. Non-static methods have "this" reference by default at index 0 in local variables. 
 * ```GETFIELD``` - opcode for reading field.
 
### Assigning to field

```java
public class AssignmentStatementGenerator {

    //constructor and fields
    
    public void generate(Assignment assignment) {
        String varName = assignment.getVarName();
        Expression expression = assignment.getExpression();
        Type type = expression.getType();
        if(scope.isLocalVariableExists(varName)) {
            int index = scope.getLocalVariableIndex(varName);
            methodVisitor.visitVarInsn(type.getStoreVariableOpcode(), index);
            return;
        }
        Field field = scope.getField(varName);
        String descriptor = field.getType().getDescriptor();
        methodVisitor.visitVarInsn(Opcodes.ALOAD,0);
        expression.accept(expressionGenerator);
        methodVisitor.visitFieldInsn(Opcodes.PUTFIELD,field.getOwnerInternalName(),field.getName(),descriptor);
    }
```

The local variables have priorority over fields if there is ambiguity. If you declared 
local variable named exactly like a field you wouldn't want to 
reference field but a variable, right? That is why the local variables are first searched.

The ```PUTFIELD``` opcode is similar to ```GETFIELD``` but pops additional item of the stack - 
the result of expression to be saved into field.

## Example


Following Enkel class:

```java
Fields {

    int field

    start {
        field = 5
        print field
    }
}
```

generates bytecode:

```bash
kuba@kuba-laptop:~/repos/Enkel-JVM-language$ javap -c Fields
public class Fields {
  public int field;

  public void start();
    Code:           
       0: aload_0               //get "this"
       1: ldc           #9      // load constant "5" from constant pool 
       3: putfield      #11     // Field field:I - pop 5 off the stack and write to field
       6: getstatic     #17     // Field java/lang/System.out:Ljava/io/PrintStream; 
       9: aload_0               //get "this" reference
      10: getfield      #11     // Field field:I
      13: invokevirtual #22     // Method "Ljava/io/PrintStream;".println:(I)V
      16: return

 //autogenerated constructor and main method
}
```