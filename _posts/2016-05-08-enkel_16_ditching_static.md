---
layout: post
title: Creating JVM language [PART 16] - Ditching statics
categories: [Enkel]
tags: [enkel,jvm,asm,antlr,antlr4,antlr,java,language]
fullview: true
comments: true
---
## Sources

The project can be cloned from **[github repository](https://github.com/JakubDziworski/Enkel-JVM-language)**.  
The revision described in this post is **[c951b7b596889ba71070a33f4582a05beddab502](https://github.com/JakubDziworski/Enkel-JVM-language/tree/c951b7b596889ba71070a33f4582a05beddab502)**.

## OOP and statics 

What is the the greatest advantage of object oriented programming? 
In my opinion it is polymorphism. How do you achieve polymorphism?
By using inheritance. Can you use inheritance with statics? No, of course not.

In my opinion statics violate the object oriented concepts, and should
not be included in truly object oriented languages. Instead of using statics
objects you are way better just by using singletons.

So why would Java call itself object oriented when there are statics?
My theory is that for some historical reason they wanted C++ guys to adapt to Java quicker and "lure" as many as possible into java world.

## Switching to purely non-static world

Until last post (about object creation) all Enkel classes were purely static.
They consisted of main method and other static methods. The reason behind this
was to first implement all basic language features like variables,conditional statements,loops,
method calls, and then move to OO. The time has come to start implementing OO.

### What about main method?

All Java programs need to have **static** main method defined. The way Enkel handles this is as follows:

 * The compiler under the hood generates static main method. 
 * Inside main method it creates an object using default constructor.
 * It calls ```start``` method on the fresh new created object.
 * A programmer provide ```start``` method.
 
 
```java
private Function getGeneratedMainMethod() {
     FunctionParameter args = new FunctionParameter("args", BultInType.STRING_ARR, Optional.empty());
     FunctionSignature functionSignature = new FunctionSignature("main", Collections.singletonList(args), BultInType.VOID);
     ConstructorCall constructorCall = new ConstructorCall(scope.getClassName());
     FunctionSignature startFunSignature = new FunctionSignature("start", Collections.emptyList(), BultInType.VOID);
     FunctionCall startFunctionCall = new FunctionCall(startFunSignature, Collections.emptyList(), scope.getClassType());
     Block block = new Block(new Scope(scope), Arrays.asList(constructorCall,startFunctionCall));
     return new Function(functionSignature, block);
 }
```

The start method is basically non-static version of main method.

### INVOKESTATIC vs INVOKEVIRTUAL

In [Creating JVM language [PART 7] - Methods](http://jakubdziworski.github.io/enkel/2016/04/05/enkel_7_methods.html) 
I used ```INVOKESTATIC``` for invoking methods. It's time to change it to **```INVOKEVIRTUAL```**.

There is one important difference between both of them - **```INVOKEVIRTUAL``` requires owner**.
```INVOKESTATIC``` pops arguments off the stack. ```INVOKEVIRTUAL``` pops owner off the stack
and then it pops arguemnts. It's mandatory to generate owner expression.

If there is no owner provided by a programmer the implicit "this" var reference is provided:

```java

//Mapping antlr generated FunctionCallContext to FunctionCall 
@Override
public Expression visitFunctionCall(@NotNull EnkelParser.FunctionCallContext ctx) {
    //other stuff
    boolean ownerIsExplicit = ctx.owner != null;
    if(ownerIsExplicit) {
        Expression owner = ctx.owner.accept(this);
        return new FunctionCall(signature, arguments, owner);
    }
    ClassType thisType = new ClassType(scope.getClassName());
    return new FunctionCall(signature, arguments, new VarReference("this",thisType)); //pass "this" as a owner 
}
```



```java
//Generating bytecode using mapped FunctionCall object
public void generate(FunctionCall functionCall) {
    functionCall.getOwner().accept(this); //generate owner (pushses it onto stack)
    generateArguments(functionCall);  //generate arguments
    String functionName = functionCall.getIdentifier();
    String methodDescriptor = DescriptorFactory.getMethodDescriptor(functionCall.getSignature());
    String ownerDescriptor = functionCall.getOwnerType().getInternalName();
    //Consumes owner and arguments off the stack
    methodVisitor.visitMethodInsn(Opcodes.INVOKEVIRTUAL, ownerDescriptor, functionName, methodDescriptor, false); 
}
```

## Example

Following Enkel Class: 

```java
HelloStart {

    start {
        print "Hey I am non-static 'start' method"
    }
}
```

get's compiled into:

```bash
kuba@kuba-laptop:~/repos/Enkel-JVM-language$ javap -c HelloStart.class 
public class HelloStart {
  public void start();
    Code:
       0: getstatic     #12                 // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #14                 // String Hey I am non-static  'start' method
       5: invokevirtual #19                 // Method "Ljava/io/PrintStream;".println:(Ljava/lang/String;)V
       8: return

  //Constructor
  public HelloStart();
    Code:
       0: aload_0   //get "this"
       1: invokespecial #22                 // Method java/lang/Object."<init>":()V - call super
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class HelloStart - create new object
       3: dup       //duplicate new object so that invokespecial does not consumes it
       4: invokespecial #25                 // Method "<init>":()V - call constructor
       7: invokevirtual #27                 // Method start:()V
      10: return
}
```

where Java's equivalent would be:

```java
public class HelloStart {
    public HelloStart() {
    }

    public static void main(String[] var0) {
        (new HelloStart()).start();
    }
    
    public void start() {
        System.out.println("Hey I am non-static \'start\' method");
    }

}
```
