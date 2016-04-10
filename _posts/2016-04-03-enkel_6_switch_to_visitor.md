---
layout: post
title: Creating JVM language [PART 6] - Switching to visitor oriented parsing
categories: [Enkel]
tags: [enkel,jvm,asm,antlr,antlr4,antlr,java,language]
fullview: true
comments: true
---
## Sources

The project can be cloned from **[github repository](https://github.com/JakubDziworski/Enkel-JVM-language)**.  
The revision described in this post is **[1fc8131b2752e73776e91084ffeabbfa45fc6307](https://github.com/JakubDziworski/Enkel-JVM-language/tree/1fc8131b2752e73776e91084ffeabbfa45fc6307)**.

## Visitor vs Listener

Previously I used listener pattern to implement Enkel parser.
There is hovewer another way to do that - Visitor. To enabled it specify ```-visitor``` on the commnad line.

I was kind of curious which one would be more suitable for Enkel so I created a small project that exposes the differences.
Check out **[this blog post ](http://jakubdziworski.github.io/java/2016/04/01/antlr_visitor_vs_listener.html)** where you can read full comparison and get sources.

The main benefit is that Visitor returns value where Listener does not:

* There is less code to write.
* Less bug prone. No need to store parsing result in the field and rely on getter.


```java
//Listener
class ClassListener extends EnkelBaseListener<ClassDeclaration> {

        private Class parsedClass;

        @Override
        public void enterClassDeclaration(@NotNull EnkelParser.ClassDeclarationContext ctx) {
            String className = ctx.className().getText();
            //do some other stuff
            parsedClass = new Class(className,methods);
        }

        public Class getParsedClass() {
            return parsedClass;
        }
    }
```

```java
//Visitor
public class ClassVisitor extends EnkelBaseVisitor<ClassDeclaration> {

    @Override
    public ClassDeclaration visitClassDeclaration(@NotNull EnkelParser.ClassDeclarationContext ctx) {
        String name = ctx.className().getText();
        //do some other stuff
        return new ClassDeclaration(name, methods);
    }
}
```

The decision to switch to Visitor pattern was therefore quite obvious.