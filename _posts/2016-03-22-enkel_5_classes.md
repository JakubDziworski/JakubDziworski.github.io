---
layout: post
title: Creating JVM language [PART 5] - Adding 'class' scope
categories: [Enkel]
tags: [enkel,jvm,asm,antlr,antlr4,antlr,java,language]
fullview: true
comments: true
---
## Sources

The project can be cloned from **[github repository](https://github.com/JakubDziworski/Enkel-JVM-language)**.  
The revision described in this post is **[50e6996a4faf8d5b469d291a029be05f9e6c9520](https://github.com/JakubDziworski/Enkel-JVM-language/tree/0f900ef537e23a15de2a100fb1e3942b7d079b36)**.

## Parser rules changes

In previous post I mentioned few things I would like to add to the language.
The first one is going to be obviously a class scope.  

Modification made to the language parsing rules:

```java
compilationUnit : ( variable | print )* EOF;
variable : VARIABLE ID EQUALS value;
```

has been changed to: 

```java
compilationUnit : classDeclaration EOF ; //root rule - our code consist consist only of variables and prints (see definition below)
classDeclaration : className '{' classBody '}' ;
className : ID ;
classBody :  ( variable | print )* ;
```

 * The file must consist of one and only one classDeclaration.
 * class declaration consist of className followed by body inside curly brackets
 * the body is the same thing as it used to be in prototype - variable declaration or prints
 
This is how the parse tree looks after modifications: 

![Parse Tree](/assets/media/enkel_5/class_parse_tree.gif)

## Compiler changes

Most of the changes involve moving top-level code from ```ByteCodeGenerator``` class to ```CompilationUnit``` ```ClassDeclaration```.
The logic is as follows:

 1. Compiler grabs parse tree values from ```SyntaxParseTreeTraverser```: 
 2. Compiler instantiates CompilationUnit:
 
    ```java 
    //Compiler.java
    final CompilationUnit compilationUnit = new SyntaxTreeTraverser().getCompilationUnit(fileAbsolutePath);
    //Check getCompilationUnit() method body on github
    ```
 3. CompilationUnit instantiates ClassDeclaration (passes class name, and instructions list)
 4. ClassDeclaration executes class specific instructions and loops over ClassScopeInstructions:

    ```java
     //ClassDeclaration.java
     MethodVisitor mv = classWriter.visitMethod(ACC_PUBLIC + ACC_STATIC, "main", "([Ljava/lang/String;)V", null, null);
     instructions.forEach(classScopeInstruction -> classScopeInstruction.apply(mv));
    ```
 
![Parse Tree](/assets/media/enkel_5/uml.png)

One additional thing that changes is that the output .class file will have name
based on the class declaration regardless of input *.enk filename:

```java
String className = compilationUnit.getClassName();
String fileName = className + ".class";
OutputStream os = new FileOutputStream(fileName);
IOUtils.write(byteCode,os);
```
