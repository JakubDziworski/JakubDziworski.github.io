---
layout: post
title: Creating JVM language [PART 17] - Referencing other classes (including Java API)
categories: [Enkel]
tags: [enkel,jvm,asm,antlr,antlr4,antlr,java,language]
fullview: true
comments: true
---
## Sources

The project can be cloned from **[github repository](https://github.com/JakubDziworski/Enkel-JVM-language)**.  
The revision described in this post is **[d0b8a3d711d9fd46f675d03ec4b506fbcb74ae22](https://github.com/JakubDziworski/Enkel-JVM-language/tree/d0b8a3d711d9fd46f675d03ec4b506fbcb74ae22)**.

## Bytecode - JVM languages common denominator

All JVM languages are compiled into bytecode, which is interpreted by the virtual machine.
This means that compilers do not know what language the referencing classes are compiled from. As long as the class is on
classpath it can be used, regardless of programming language.

This opens huge possibilities. All Java libraries,utilities and frameworks
can now be used by Enkel.

## Finding classes methods and constructors  

When you reference class defined in different class file you have two choices:

 * Runtime - trust programmer and generate bytecode without verifying that a signature exists on a classpath. 
 This will throw **exceptions at runtime** if signature is not available on classpath.
 * Compile time - verify that signature exists on classpath before generating bytecode. This will stop
 compilation process if some referenced signature is not available.
 
 In Enkel I decided to go with the second option - mainly due to safety reasons.
 It can be achieved using reflection api:
 
```java
public class ClassPathScope {

 public Optional<FunctionSignature> getMethodSignature(Type owner, String methodName, List<Type> arguments) {
     try {
         Class<?> methodOwnerClass = owner.getTypeClass();
         Class<?>[] params = arguments.stream()
                 .map(Type::getTypeClass).toArray(Class<?>[]::new);
         Method method = methodOwnerClass.getMethod(methodName,params);
         return Optional.of(ReflectionObjectToSignatureMapper.fromMethod(method));
     } catch (Exception e) {
         return Optional.empty();
     }
 }

 public Optional<FunctionSignature> getConstructorSignature(String className, List<Type> arguments) {
     try {
         Class<?> methodOwnerClass = Class.forName(className);
         Class<?>[] params = arguments.stream()
                 .map(Type::getTypeClass).toArray(Class<?>[]::new);
         Constructor<?> constructor = methodOwnerClass.getConstructor(params);
         return Optional.of(ReflectionObjectToSignatureMapper.fromConstructor(constructor));
     } catch (Exception e) {
         return Optional.empty();
     }
 }
}
```

If the method (or constructor) is not found then the exception is thrown and 
the compilation process is terminated:

```java
    //Scope.java
    return new ClassPathScope().getMethodSignature(owner.get(), methodName, argumentsTypes)
                    .orElseThrow(() -> new MethodSignatureNotFoundException(this,methodName,arguments));
```

This approach seems safer, but is also slower. All the dependencies must be resolved
while compiling using expensive reflection.

## Examples

### Calling other Enkel classes 

Let's try to call ```Library``` class from ```Client``` class:

```java
 Client {
 
     start {
         print "Client: Calling my own 'Library' class:"
         var myLibrary = new Library()
         var addition = myLibrary.add(5,2)
         print "Client: Result returned from 'Library.add' = " + addition
     }
 
 }
```

```java
Library {

    int add(int x,int y) {
        print "Library: add() method called"
        return x+y
    }

}
```

First we need to compile ```Library``` (no multiple files compilation is supported so far).
If we did not do this the ```Client``` compilation would fail due to unresolved reference
to ```Library``` class.


```bash
kuba@kuba-laptop:~/repos/Enkel-JVM-language$ java -classpath compiler/target/compiler-1.0-SNAPSHOT-jar-with-dependencies.jar:. com.kubadziworski.compiler.Compiler EnkelExamples/ClassPathCalls/Library.enk 
kuba@kuba-laptop:~/repos/Enkel-JVM-language$ java -classpath compiler/target/compiler-1.0-SNAPSHOT-jar-with-dependencies.jar:. com.kubadziworski.compiler.Compiler EnkelExamples/ClassPathCalls/Client.enk 
kuba@kuba-laptop:~/repos/Enkel-JVM-language$ java Client 
Client: Calling my own 'Library' class:
Library: add() method called
Client: Result returned from 'Library.add' = 7
```

### Calling Java API!

```java
Client {

    start {
        var someString = "someString"
        print someString + " to upper case : " +  someString.toUpperCase()
    }

}
```

```bash
kuba@kuba-laptop:~/repos/Enkel-JVM-language$ java Client 
cos to upper case = COS
```

