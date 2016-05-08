---
layout: post
title: Creating JVM language [PART 15] - Constructors
categories: [Enkel]
tags: [enkel,jvm,asm,antlr,antlr4,antlr,java,language]
fullview: true
comments: true
---
## Sources

The project can be cloned from **[github repository](https://github.com/JakubDziworski/Enkel-JVM-language)**.  
The revision described in this post is **[c951b7b596889ba71070a33f4582a05beddab502](https://github.com/JakubDziworski/Enkel-JVM-language/tree/c951b7b596889ba71070a33f4582a05beddab502)**.

## Syntax

Enkel's constuctor declaration and invocation syntax are the same as Java's (well except features like default and named parameters).

Declaration Example:

```java
Cat ( String name ) {
}
```

Invocation:

```java
new Cat ( "Molly" ) 
```

## Grammar Changes

In Java the constructor declaration syntax is just a method declaration without return type.
Turns out Enkel method declarations do not require specifing return value (if the method returns void).
Therefore there is no need to specify new rule for constructor declaration. 

But what about constructor call? How would a parser distinct between method call and constructor call?
For that reason Enkel introduces 'new' keyword:

```java
//other rules
expression : //other rules alternatives
           | 'new' className '('argument? (',' argument)* ')' #constructorCall
```           

## Mapping antlr context objects

The introduction of new rule alternative (constructorCall) leads to a new parse tree callback:

```java
@Override
public Expression visitConstructorCall(@NotNull EnkelParser.ConstructorCallContext ctx) {
    String className = ctx.className().getText();
    List<EnkelParser.ArgumentContext> argumentsCtx = ctx.argument();
    List<Expression> arguments = getArgumentsForCall(argumentsCtx, className);
    return new ConstructorCall(className, arguments);
}
```

FunctionCall requires name,return type,arguments and owner expression provided for constructor.
Constructor call however requires only className and arguments:

 * Does a constructor need return type provided? No - it's always the same - the type of the class it is defined in.
 * Does a constructor need owner expression? No - it's always called with a 'new' keyword. The statement like
```someObject.new SomeObject()``` would not make any sense.


What about Constructor declaration? There is no rule alternative for it.
How do we then distinguish between function declaration and constructor?
Simply by checking if the name of the method is equal to the current Class name.
It also means that regular methods cannot be named like a class:

```java
@Override
    public Function visitFunction(@NotNull EnkelParser.FunctionContext ctx) {
        List<Type> parameterTypes = ctx.functionDeclaration().functionParameter().stream()
                .map(p -> TypeResolver.getFromTypeName(p.type())).collect(toList());
        FunctionSignature signature = scope.getMethodCallSignature(ctx.functionDeclaration().functionName().getText(),parameterTypes);
        scope.addLocalVariable(new LocalVariable("this",scope.getClassType()));
        addParametersAsLocalVariables(signature);
        Statement block = getBlock(ctx);
        //Check if method is not actually a constructor
        if(signature.getName().equals(scope.getClassName())) {
            return new Constructor(signature,block);
        }
        return new Function(signature, block);
    }
```

### Default constructor?

Enkel also creates default constructor if you do not provide one:

```java
@Override
public ClassDeclaration visitClassDeclaration(@NotNull EnkelParser.ClassDeclarationContext ctx) {
    //some other stuff
    boolean defaultConstructorExists = scope.parameterLessSignatureExists(className);
    addDefaultConstructorSignatureToScope(name, defaultConstructorExists);
    //other stuff
    if(!defaultConstructorExists) {
        methods.add(getDefaultConstructor());
    }
}
        
private void addDefaultConstructorSignatureToScope(String name, boolean defaultConstructorExists) {
    if(!defaultConstructorExists) {
        FunctionSignature constructorSignature = new FunctionSignature(name, Collections.emptyList(), BultInType.VOID);
        scope.addSignature(constructorSignature);
    }
}

private Constructor getDefaultConstructor() {
    FunctionSignature signature = scope.getMethodCallSignatureWithoutParameters(scope.getClassName());
    Constructor constructor = new Constructor(signature, Block.empty(scope));
    return constructor;
}
```

You may wonder why the constructor returns void. Roughlt speaking JVM divides object creation into two steps - first it allocates it,
then it calls constructor (which responsibility is to initialize already created object). Thanks to that you can call "this" inside constructors.

## Generating bytecode

We've got constructor declarations and invocations properly parsed
and mapped to nice objects representing them. How to reach out 
data from them and generate bytecode?

Object creation in jvm bytecode is divided into two instruction:

 * ```NEW``` - allocates memory on the heap, initialize instance members to a default values (int - 0, boolean - false etc.)
 * ```INVOKESPECIAL``` - calls constructor 

In Java you do not need to call super() in the constructor, right?
It is required - if you do not do this the java compiler does it automatically. 
The object cannot be created without calling super!

Invoking a super call happens using ```INVOKESPECIAL```, and the Enkel compiler
handles it automatically (similarly to java compiler).

### Generating bytecode for constructor call

```java
public void generate(ConstructorCall constructorCall) {
        String ownerDescriptor = scope.getClassInternalName(); //example : java/lang/String
        methodVisitor.visitTypeInsn(Opcodes.NEW, ownerDescriptor); //NEW instruction takes object decriptor as an input
        methodVisitor.visitInsn(Opcodes.DUP); //Duplicate (we do not want invokespecial to "eat" our brand new object
        FunctionSignature methodCallSignature = scope.getMethodCallSignature(constructorCall.getIdentifier(),constructorCall.getArguments());
        String methodDescriptor = DescriptorFactory.getMethodDescriptor(methodCallSignature);
        generateArguments(constructorCall);
        methodVisitor.visitMethodInsn(Opcodes.INVOKESPECIAL, ownerDescriptor, "<init>", methodDescriptor, false);
    }
```

You may wonder why do we need DUP instruction?
After NEW instruction has been called the stack contains brand new created object.
INVOKESPECIAL pops element (object) from the stack to initialize it.
If we didn't duplicate the object it would just be popped by constructor,
and lost in the heap waited for GC to collect it.

The following statement:

```java
new Cat().meow()
```

is then compiled into bytecode:

```java
0: new           #2                  // class Cat
3: dup
4: invokespecial #23                 // Method "<init>":()V
7: invokevirtual #26                 // Method meow:()V
```

### Generating bytecode for constructor declaration

```java
public void generate(Constructor constructor) {
    Block block = (Block) constructor.getRootStatement();
    Scope scope = block.getScope();
    int access = Opcodes.ACC_PUBLIC;
    String description = DescriptorFactory.getMethodDescriptor(constructor);
    MethodVisitor mv = classWriter.visitMethod(access, "<init>", description, null, null);
    mv.visitCode();
    StatementGenerator statementScopeGenrator = new StatementGenerator(mv,scope);
    new SuperCall().accept(statementScopeGenrator); //CALL SUPER IMPLICITILY BEFORE BODY ITSELF
    block.accept(statementScopeGenrator); //CALL THE BODY DEFINED BY PROGRAMMER
    appendReturnIfNotExists(constructor, block,statementScopeGenrator);
    mv.visitMaxs(-1,-1);
    mv.visitEnd();
}
```

As I mentioned above the super call is required to be the very first expression in every constructor.
As a Java programmers we usually do not specify it (unless the superclass does not have parameterless constructor).
It is not because it is not required - the java compiler generates it automatically.
It would be awesome if Enkel compiler could do the same:

```java
new SuperCall().accept(statementScopeGenrator); 
```
invokes 

```java
public void generate(SuperCall superCall) {
    methodVisitor.visitVarInsn(Opcodes.ALOAD,0); //LOAD "this" object
    generateArguments(superCall);
    String ownerDescriptor = scope.getSuperClassInternalName();
    methodVisitor.visitMethodInsn(Opcodes.INVOKESPECIAL, ownerDescriptor, "<init>", "()V" , false);
}
```

Every method (even constructor) treats arguments as local variables in the frame.
If the method ```int add(int x,int y)``` was called in a static context
then its initial frame would consisted of 2 variables (x,y). 
Additionally if the method is non-static it also puts ```this``` object (the object on which it was called)
in the frame (at position 0). So if the ```add``` method was called in a non-static context
it would have 3 local variables (this,x,y) out of the box.

The Cat constructor without any body specified by programmer would therefore look like:

```java
0: aload_0      //load "this"
1: invokespecial #8                  // Method java/lang/Object."<init>":()V - call super on "this" (the Cat dervies from Object)
12: return
```
 

