---
layout: post
title: Creating JVM language [PART 7] - Methods
categories: [Enkel]
tags: [enkel,jvm,asm,antlr,antlr4,antlr,java,language]
fullview: true
comments: true
---
## Sources

The project can be cloned from **[github repository](https://github.com/JakubDziworski/Enkel-JVM-language)**.  
The revision described in this post is **[1fc8131b2752e73776e91084ffeabbfa45fc6307](https://github.com/JakubDziworski/Enkel-JVM-language/tree/1fc8131b2752e73776e91084ffeabbfa45fc6307)**.


## Methods

So far It is was possible to declare class and variables within one global scope.
Next step is creating methods.

The goal is to compile following Enkel class:

```java
First {
    void main (string[] args) {
        var x = 25
        metoda(x)
    }

    void metoda (int param) {
        print param
    }
}
```

## Scope
To acess other functions and variables they need to be in the scope:
    
```java
public class Scope {
    private List<Identifier> identifiers; //think of it as a variables for now
    private List<FunctionSignature> functionSignatures;
    private final MetaData metaData;  //currently stores only class name

    public Scope(MetaData metaData) {
        identifiers = new ArrayList<>();
        functionSignatures = new ArrayList<>();
        this.metaData = metaData;
    }

    public Scope(Scope scope) {
        metaData = scope.metaData;
        identifiers = Lists.newArrayList(scope.identifiers);
        functionSignatures = Lists.newArrayList(scope.functionSignatures);
    }
    
    //some other methods that expose data to the outside
}         

```
The scope object is created during class creation and passed to the children (functions).
Children copy Scope (using one of the constructors) and add some other items to it.

## Signatures

When calling a method there must be some kind of information about it available.
Let's say you have the following psudocode:

```java
f1() {
    f2()
}

f2(){
}
```

Which results in following parse tree:

<div class="mermaid" style="">
graph TD;
    Root-->Function:f1;
    Root-->Function:f2;
    Function:f1-->FunctionCall:f2;
</div>

Nodes are visited in following order follow:

* Root
* Function:f1
* FunctionCall:f2 //ERROR! f2??! What is that? It's not yet been declared!! 
* Function:f2

So the problem is that during method invokation the method might not yet been visited. There is no information about f2 during parsing f1!

To fix that problem it is mandatory to visit all Method Declarations and store their signatures in the scope:

```java
public class ClassVisitor extends EnkelBaseVisitor<ClassDeclaration> {

 private Scope scope;

 @Override
 public ClassDeclaration visitClassDeclaration(@NotNull EnkelParser.ClassDeclarationContext ctx) {
     String name = ctx.className().getText();
     FunctionSignatureVisitor functionSignatureVisitor = new FunctionSignatureVisitor();
     List<EnkelParser.FunctionContext> methodsCtx = ctx.classBody().function();
     MetaData metaData = new MetaData(ctx.className().getText());
     scope = new Scope(metaData);
     //First find all signatures
     List<FunctionSignature> signatures = methodsCtx.stream()
             .map(method -> method.functionDeclaration().accept(functionSignatureVisitor))
             .peek(scope::addSignature)
             .collect(Collectors.toList());
     //Once the signatures are found start parsing methods
     List<Function> methods = methodsCtx.stream()
             .map(method -> method.accept(new FunctionVisitor(scope)))
             .collect(Collectors.toList());
     return new ClassDeclaration(name, methods);
 }
}
```

## Invokestatic

Once all the information about the codes has been parsed It is time to convert it to bytecode.
Since I haven not yet implemented object creation, methods need to be called in a static context.

```java 

int access = Opcodes.ACC_PUBLIC + Opcodes.ACC_STATIC;
```
The bytecode instruction for static method invokation is called  **```invokestatic```**.
This instruction has two parameters which require:

 * [field descriptor](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.3.2) of the class that has the method (Ljava/io/PrintStream;)
 * [method descriptor](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.3.3)  (example: println:(I)V)

Values from operand stack are assumed to be parameters (amount and type must match method descriptor).

```java


public class MethodGenerator {
    private final ClassWriter classWriter;

    public MethodGenerator(ClassWriter classWriter) {
        this.classWriter = classWriter;
    }

    public void generate(Function function) {
        Scope scope = function.getScope();
        String name = function.getName();
        String description = DescriptorFactory.getMethodDescriptor(function);
        Collection<Statement> instructions = function.getStatements();
        int access = Opcodes.ACC_PUBLIC + Opcodes.ACC_STATIC;
        MethodVisitor mv = classWriter.visitMethod(access, name, description, null, null);
        mv.visitCode();
        StatementGenerator statementScopeGenrator = new StatementGenerator(mv);
        instructions.forEach(instr -> statementScopeGenrator.generate(instr,scope));
        mv.visitInsn(Opcodes.RETURN);
        mv.visitMaxs(-1,-1); //asm autmatically calculate those but the call is required
        mv.visitEnd();
    }
}
```
## Results

The following Enkel code:

```java
First {
    void main (string[] args) {
        var x = 25
        metoda(x)
    }

    void metoda (int param) {
        print param
    }
}
```

gets compiled into following bytecode:

```java
kuba@kuba-laptop:~/repos/Enkel-JVM-language$ javap -c First
public class First {
  public static void main(java.lang.String[]);
    Code:
       0: bipush        25 //push value 25 onto the stack
       2: istore_0         //store value from stack into variable at index 0
       3: iload_0          //load variable at index onto the stack
       5: invokestatic  #10 //call metod Method metoda:(I)V  
       8: return

  public static void metoda(int);
    Code:
       0: getstatic     #16                 // Field java/lang/System.out:Ljava/io/PrintStream;
       3: iload_0
       4: invokevirtual #20                 // Method "Ljava/io/PrintStream;".println:(I)V
       7: return
}
```

