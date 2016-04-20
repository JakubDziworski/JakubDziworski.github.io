---
layout: post
title: Creating JVM language [PART 2] - Minimum theory
categories: [Enkel]
description: First post.
tags: [enkel,jvm,java]
fullview: true
comments: true
---
At the very top level of abstraction I will need to implement three modules.

| Module | Takes | Returns |
| ------------- |-------------| -----|
| Lexer | Text (Code) | Tokens |
| Parser | Tokens | AST (Abstract Syntax Tree) |
| Compiler | AST | JVM Bytecode |

###Lexer
Lexer takes simple text input and tokenizes it. The code is no longer a meaningless stream of bytes, but a  list of tokens. Tokens are also associated with type useful for further analysis.
###Parser
The tokens are passed to parser which is responsible for organizaing tokens into hierarchical structure called **Abstract Syntax Tree**. The tree determines the order in which code should be executed.
###Compiler
Compiler traverses the tree and maps it into valid bytecode instructions.

##Example

Let's assume I'd like to execute ```int x=a*5+2;``` expression. The following steps need to be taken:
<div class="mermaid" style="">
graph LR
        A["
        Input Code<br><br>
        int x=a*5+2;
        "]
        B["
          Tokens<br><br>{type,int},
          <br>{identifier,x}
          <br>{operator,=}
          <br>{identifier,a}
          <br>{operator,#42;}
          <br>{number,5}
          <br>{operator,+}
          <br>{number,2}
          <br>{keyword,;}
          "]
          A-->|LEXER|B
          B-->|PARSER|EQUALS
        subgraph Abstract Syntax Tree
        EQUALS["="]
        VARX["x"]
        VARA["a"]
        MULTIPLY["#42;"]
        PLUS["+"]
        FIVE[5]
        TWO[2]
        EQUALS---PLUS
        EQUALS---VARX
        PLUS---TWO
        PLUS---MULTIPLY
        MULTIPLY---FIVE
        MULTIPLY---VARA
        end
</div>
Once the abstract tree is created it needs to be mapped to bytecode by compiler.
