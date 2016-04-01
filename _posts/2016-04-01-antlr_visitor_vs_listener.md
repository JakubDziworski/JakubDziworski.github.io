---
layout: post
title: Antlr4 - Listener vs Visitor implementation 
categories: [Java]
tags: [enkel,jvm,antlr,antlr4,antlr,java,language]
fullview: true
comments: true
---
## Sources

The project source code can be cloned from https://github.com/JakubDziworski/AntlrListenerVisitorComparison.
There are full example of Listener and Visitor oriented parser implementations.

## SomeLanguage

Let's say we want to parse "SomeLanguage" with following grammar.

```java
grammar SomeLanguage ;

classDeclaration : 'class' className '{' (method)* '}';
className : ID ;
method : methodName '{' (instruction)+ '}' ;
methodName : ID ;
instruction : ID ;

ID : [a-zA-Z0-9]+ ;
WS: [ \t\n\r]+ -> skip ;
```

Sample valid "SomeLanguage" code:
```java 
class SomeClass {
    fun1 {
        instruction11
        instruction12
    }
    fun2 {
        instruction21
        instruction22
    }
};
```

The class consist of zero or more methods.
The methods consist of instructions.
That's all.

We'd like to parse the code to "Class" object:

```java 
public class Class {
    private String name;
    private Collection<Method> methods;
}

public class Method {
    private String name;
    private Collection<Instruction> instructions;
}

public class Instruction {
    private String name;
}
```

## Listener vs Visitor

To do that Antlr4 provides two ways of traversing syntax tree:

 * Listener (default)
 * Visitor

To generate visitor classes from the grammar file you have to add ```-visitor``` option to the command line. I however use antlr maven plugin (see full code at github)


## Parsing using Listener 

To parse the code to Class object  we could create one big Listener and
register it using parser (parser.addParseListener()).
This is however going to generete one huge messy class.
Instead it is good idea to register separate listener for each
rule separetly:


```java 
public class ListenerOrientedParser implements Parser{

    @Override
    public Class parse(String code) {
        CharStream charStream = new ANTLRInputStream(code);
        SomeLanguageLexer lexer = new SomeLanguageLexer(charStream);
        TokenStream tokens = new CommonTokenStream(lexer);
        SomeLanguageParser parser = new SomeLanguageParser(tokens);

        ClassListener classListener = new ClassListener();
        parser.classDeclaration().enterRule(classListener);
        return classListener.getParsedClass();
    }

    class ClassListener extends SomeLanguageBaseListener {

        private Class parsedClass;

        @Override
        public void enterClassDeclaration(@NotNull SomeLanguageParser.ClassDeclarationContext ctx) {
            String className = ctx.className().getText();
            MethodListener methodListener = new MethodListener();
            ctx.method().forEach(method -> method.enterRule(methodListener));
            Collection<Method> methods = methodListener.getMethods();
            parsedClass = new Class(className,methods);
        }

        public Class getParsedClass() {
            return parsedClass;
        }
    }

    class MethodListener extends SomeLanguageBaseListener {

        private Collection<Method> methods;

        public MethodListener() {
            methods = new ArrayList<>();
        }

        @Override
        public void enterMethod(@NotNull SomeLanguageParser.MethodContext ctx) {
            String methodName = ctx.methodName().getText();
            InstructionListener instructionListener = new InstructionListener();
            ctx.instruction().forEach(instruction -> instruction.enterRule(instructionListener));
            Collection<Instruction> instructions = instructionListener.getInstructions();
            methods.add(new Method(methodName, instructions));
        }

        public Collection<Method> getMethods() {
            return methods;
        }
    }

    class InstructionListener extends SomeLanguageBaseListener {

        private Collection<Instruction> instructions;

        public InstructionListener() {
            instructions = new ArrayList<>();
        }

        @Override
        public void enterInstruction(@NotNull SomeLanguageParser.InstructionContext ctx) {
            String instructionName = ctx.getText();
            instructions.add(new Instruction(instructionName));
        }

        public Collection<Instruction> getInstructions() {
            return instructions;
        }
    }
}
```

## Parsing using Visitor

Visitor implementation is very similar but have one advantage. **Visitor methods return value - no need to store values in fields**.
  
```java
public class VisitorOrientedParser implements Parser {

    public Class parse(String someLangSourceCode) {
        CharStream charStream = new ANTLRInputStream(someLangSourceCode);
        SomeLanguageLexer lexer = new SomeLanguageLexer(charStream);
        TokenStream tokens = new CommonTokenStream(lexer);
        SomeLanguageParser parser = new SomeLanguageParser(tokens);

        ClassVisitor classVisitor = new ClassVisitor();
        Class traverseResult = classVisitor.visit(parser.classDeclaration());
        return traverseResult;
    }

    private static class ClassVisitor extends SomeLanguageBaseVisitor<Class> {
        @Override
        public Class visitClassDeclaration(@NotNull SomeLanguageParser.ClassDeclarationContext ctx) {
            String className = ctx.className().getText();
            MethodVisitor methodVisitor = new MethodVisitor();
            List<Method> methods = ctx.method()
                    .stream()
                    .map(method -> method.accept(methodVisitor))
                    .collect(toList());
            return new Class(className, methods);
        }
    }

    private static class MethodVisitor extends SomeLanguageBaseVisitor<Method> {
        @Override
        public Method visitMethod(@NotNull SomeLanguageParser.MethodContext ctx) {
            String methodName = ctx.methodName().getText();
            InstructionVisitor instructionVisitor = new InstructionVisitor();
            List<Instruction> instructions = ctx.instruction()
                    .stream()
                    .map(instruction -> instruction.accept(instructionVisitor))
                    .collect(toList());
            return new Method(methodName, instructions);
        }
    }

    private static class InstructionVisitor extends  SomeLanguageBaseVisitor<Instruction> {

        @Override
        public Instruction visitInstruction(@NotNull SomeLanguageParser.InstructionContext ctx) {
            String instructionName = ctx.getText();
            return new Instruction(instructionName);
        }
    }
}
```

## Results

Both implementations output the same result. I personally prefer
Visitor since it requires less code and there is no need to store values
in fields.

Using any parser implementation "SomeLanguage" sample code is parsed to Class object: 

```json
{
    "name": "SomeClass",
    "methods": [
        {
            "name": "fun1",
            "instructions": [
                {
                    "name": "instruction11"
                },
                {
                    "name": "instruction12"
                }
            ]
        },
        {
            "name": "fun2",
            "instructions": [
                {
                    "name": "instruction21"
                },
                {
                    "name": "instruction22"
                }
            ]
        }
    ]
}
```

For full code visit https://github.com/JakubDziworski/AntlrListenerVisitorComparison.