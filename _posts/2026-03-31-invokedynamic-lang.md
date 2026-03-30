---
title: "JVM Bites 1: Implementing a Dynamically Typed Language for the JVM"
layout: post
published: true
summary: Explore the JVM's invokedynamic machinery in this blogpost.
---
*Hopefully, this is a start of a new series, where I describe internal workings of the JVM :)*

TLDR; *Have you ever wondered how a language like Groovy or Clojure evaluates an expression like `A+B`, when it doesn't know at compile time whether to do integer addition or string concatenation or something else. How does it do this efficiently on a JVM that was built for statically typed Java? The answer is the `invokedynamic` bytecode. In this post, I build a tiny language that faces the same problem and show how to solve it using the same machinery*.

**The code snippets in this blogpost are taken from [here](https://github.com/Nirhar/dyn-typed-lang-using-indy)**

The JVM is a versatile piece of software, that executes Java Bytecode. If you are an engineer, interested in writing your own language, targeting it to the JVM seems like a good solution. The JVM ships with a Garbage Collector, thus, you need not ask your language users to manage memory. It usually also ships with an optimizing compiler and profiler, that allows you to profile and *JIT(Just in Time)*-compile user bytecode, giving you performance competitive with *AOT(Ahead of Time)* compiled languages like C++/Rust. It also gives you access to the JVM ecosystem, one that is rich in tooling (profilers, debuggers, etc), allows you to interoperate with Java libraries, etc. 

Surely, as your solution to the M:N[^mn] problem, it must be a great choice to target Java Bytecode as your IR, right? But if your language was dynamically-typed[^dyn-typing], this was not always possible. The JVM was primarily built for Java, which was a statically typed language. While it did have runtime type information available, it lacked a way to dispatch method calls based on runtime types, outside of the Java type hierarchy[^java-type-hierarchy]. This changed in 2011[^jsr292], when the *invokedynamic* bytecode was added to the JVM Specification, after which languages that were dynamically type-checked could be targeted to the JVM.


[^dyn-typing]: The point of dynamic typing was to allow programmers to not be concerned with types while programming. While this works for small projects/prototyping, big enterprise software almost always benefit from static typing.
[^java-type-hierarchy]: Java constrains your "dynamic" dispatch of method calls to the object inheritance tree that you've defined in your program.
[^mn]: In compiler design, The M:N problem refers to the inefficient solution of creating MxN compilers for M Languages and N Targets. The efficient solution is to introduce an Intermediate Representation(IR) and split the effort into M+N reusable components, i.e,  M frontends and N targets for the IR. 
[^jsr292]: invokedynamic was the main focus of [JSR-292](https://jcp.org/en/jsr/proposalDetails?id=292), that made implementing dynamic typing on top of the JVM possible.

Prior to *invokedynamic*, the JVM had four bytecodes that dealt with function calls:
1. **invokestatic**: Used to call static functions in a class
2. **invokespecial**: Used for constructors of objects, private methods and superclass method calls.
3. **invokevirtual**: Used to call member functions of objects, by resolving the call. Probably, the closest one can get to dynamic dispatch of calls, based on runtime types. However it constrains the dispatch to the inheritance tree of the classes.
4. **invokeinterface**: Similar to invokevirtual, but used to call interface methods. This is a separate instruction because invokevirtual calls can be "devirtualized" as an optimization[^devirt] but interface calls must always do an itable(*interface-table*) lookup.

And none of these bytecodes would help you implement custom dispatch logic at runtime. Their behaviour is too specific to the needs of Java, which did not require custom dynamic dispatch. Implementors of runtimes of other languages like Ruby and Python[^indy-prod-ex] were interested in targeting their languages for the JVM, but could not do so, because the JVM does not support custom dispatch logic. 

[^devirt]: A common optimization used by JIT compilers. If the compiler can prove that the runtime types are constant, or if the profiler data can allow it to speculate so, then the compiler can emit code that avoids a vtable lookup, and directly dispatch the call to the correct method.
[^indy-prod-ex]: [JRuby](https://www.jruby.org/) and [Jython](https://www.jython.org/) were attempts to use the JVM to provide a runtime for Ruby and Python. As of 2026, JRuby still remains as a modern and thriving project, whereas Jython did not move past Python 2, and is not actively developed.

*JSR-292* introduced a wild-card bytecode for function-calls, *invokedynamic* (henceforth referred to as *indy*), that can be used to implement a call, in any way you want during runtime. This bytecode can be used to do a runtime type-check and correctly dispatch a call based on the concrete types "flowing" into the call.

Okay, now that's a brief history of the `invokedynamic` bytecode. Now let's see how it can be used to implement a dynamically typed language.

# Invokedynamic machinery

The *indy* bytecode achieves custom dispatch through two key abstractions: a **MethodHandle** and a **CallSite**.

A `MethodHandle` is a typed, directly executable reference to a method — analogous to a first-class function or functor in other languages. And a `CallSite` bears a reference to a `MethodHandle` - which may be constant or mutable depending on the Callsite subtype.

Each *indy* instruction in the bytecode carries three things:

```
invokedynamic "add" : (String, String) -> String  [bootstrap = MyBootstrap.bootstrap]
```

1. A reference to a **bootstrap method** — the method the JVM should call to set up this call site
2. A **name** — a symbolic name for the call (e.g. `"add"`)
3. A **type descriptor** — the signature of the call (argument types and return type)

When an *indy* call is encountered for the first time, we make a call to the bootstrap method. It returns a CallSite object that holds a MethodHandle - the actual target to invoke. Once the bootstrap returns, the JVM links the *indy* site to the CallSite object, and on every subsequent execution of this specific *indy*, the JVM reuses the cached CallSite and invokes its MethodHandle directly. 

```
// First hit:
callsite = bootstrap(caller, name, type)
cache(indy_site → callsite)
result = callsite.getTarget().invoke(args...)

// All subsequent hits:
result = cached_callsite.getTarget().invoke(args...)
```

This is what the bootstrap method signature looks like: 

```
static CallSite bootstrap(Lookup caller, String name, MethodType type)
```

The JVM calls it automatically the **first time** the *indy* site is executed, passing:
- `caller` — a lookup context scoped to the calling class, used to find methods. Think of it as a menu of functions that you can choose from to define the behaviour of the Callsite.
- `name` — the name from the bytecode, helps to uniquely identify the *indy* from the bootstrap method.
- `type` — the method signature from the bytecode. The bootstrap method must return a `MethodHandle` whose type is compatible with this — the JVM will throw a `WrongMethodTypeException` otherwise.

Let's now look at our toy language and how we wire it up. Read on if you are really interested in the nitty-gritty!

# Our Toy Language

For the sake of this demonstration, I chose a very simple, but not so useful "language" (well, it's just the evaluation of an expression), but we just pretend that the operands of the expression have unknown types until runtime. Our language "standard" is as follows:

1. A program only consists of one binary expression, using `+`.
2. The runtime just *evaluates* the expression and prints it to console
3. Each term in the expression is a string. However, each string-term can be interpreted as numbers for arithmetic.
4. The evaluation rules are as follows:
  - If LHS and RHS can both be interpreted as numbers, then add them arithmetically. Ex: `"5" + "6" -> "11"`
  - If LHS is a Number and RHS is a String, then we throw an exception, that we cannot append strings to numbers. Ex: `"5" + "Hello!" -> Exception thrown`
  - In all other cases, we convert both LHS and RHS to strings and concatenate them with a space. Ex: `"Hello" + "5" -> "Hello 5"` and `"Hello" + "World" -> "Hello World"`   


# Implementing our toy-language

The parser for this language is written using antlr[^antlr]. The grammar file can be found [here](https://github.com/Nirhar/dyn-typed-lang-using-indy/blob/main/src/main/antlr4/com/mydynlang/MyDynLang.g4). Antlr takes care of parsing our source code, and producing an Abstract Syntax Tree, whose nodes we can *visit* to generate Java Bytecode.

[^antlr]: [Antlr](https://www.antlr.org/) is a popular parser-generator library, that can be used in multiple languages. If you want to implement a parser in Java, you should first reach out to this library.

To generate Java Bytecode, I use ObjectWeb ASM[^objweb]. We can use this library while visiting the parse-tree nodes to emit Java Bytecode. When we visit the `+` operator, we can ask ObjectWeb ASM to emit an invokedynamic bytecode:

[^objweb]: [ObjectWeb ASM](https://asm.ow2.io/) is a Java library to manipulate and analyse Java Bytecode.

```java
public Void visitExpression(MyDynLangParser.ExpressionContext ctx) {
    // Code to visit LHS of expression.
    ...
    // Code to visit RHS of expression.
    ...
    
    // Create a MethodHandle Object that holds a reference to our BootStrap method.
    // The BootStrap method is a method called `bootstrap` in a class called `MyDynLangAdd`
    Handle bootstrapHandle = new Handle(
        H_INVOKESTATIC, // Bootstrap is usually a static method
        "mydynlang/MyDynLangAdd",
        "bootstrap",
        // Java Bytecode's very verbose format to describe the Method Type.
        "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;",
        false // Not an interface
    );
    // Emit InvokeDynamic Instruction. mv is a MethodVisitor Object given by ObjectWeb ASM.
    mv.visitInvokeDynamicInsn(
        // Name to uniquely identify the callsite.
        "add", 
        // Method Signature
        "(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;", 
        // Bootstrap MethodHandle
        bootstrapHandle
    );
    
    // Code to emit bytecodes to print the result
    ...
}
```

I've omitted large chunks of boilerplate code to focus on the code around *indy*. But you can find the full Visitor implementation [here](https://github.com/Nirhar/dyn-typed-lang-using-indy/blob/main/src/main/java/mydynlang/MyDynLangVisitorImpl.java).

Now, when we give an input of `"Hi" + "5"` to our compiler, here is the Bytecode that we emit:

```
{
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=1, args_size=1
         0: getstatic     #12                 // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #14                 // String Hi
         5: ldc           #16                 // String 5
         7: invokedynamic #27,  0             // InvokeDynamic #0:add:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
        12: invokevirtual #33                 // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
        15: return
}
BootstrapMethods:
  0: #23 REF_invokeStatic mydynlang/MyDynLangAdd.bootstrap:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
```

The JVM is a stack-based interpreter[^jbyte], and the above bytecode essentially translates to this:
1. Push PrintStream object on stack. This will be needed when we print the result.
2. Push the string "Hi" on the stack
3. Push the string "5" on the stack
4. Call the bootstrap function using invokedynamic. This will set up the callsite, and redirect the call to the actual implementation. This will consume the top two stack elements and replace it with the result of the call.
5. Call Println with result of the call and the PrintStream object. This will print the result to stdout.

[^jbyte]: If you are unfamiliar with reading Java Bytecode, I'd suggest reading [this](https://foojay.io/today/java-bytecode-simplified-journey-to-the-wonderland-part-1/). 

Now that we have seen how the *indy* callsite looks, let's trace how the bootstrap method implements the dynamic call. Our bootstrap method is pretty simple, it binds the *indy* callsite to a dispatch function:

```java
// Bootstrap method
public static CallSite bootstrap(
    MethodHandles.Lookup lookup,
    String name,
    MethodType type
) throws Exception {
    MutableCallSite callsite = new MutableCallSite(type);

    // Find the dispatch function from the list of available functions. The lookup object is used for this.
    MethodHandle dispatcher = lookup.findStatic(
        MyDynLangAdd.class,
        "dispatcher",
        MethodType.methodType(
            /*Return type=*/ String.class,
            MutableCallSite.class,
            MethodHandles.Lookup.class,
            String.class,
            String.class
        )
    );
    
    // Bind the dispatch function to the callsite object
    dispatcher = dispatcher.bindTo(callsite); // Binds callsite as the first argument
    dispatcher = dispatcher.bindTo(lookup); // Binds lookup as the second argument

    callsite.setTarget(dispatcher);
    return callsite;
}
```

The dispatch function checks if the left and right arguments can be parsed as Integers and then takes appropriate action (i.e., dispatch to `addInts` or `concatStrings` or throw Exception):
```java
public static String dispatcher(
        MutableCallSite callsite,
        MethodHandles.Lookup lookup,
        String left,
        String right
    ) throws Throwable {
        Function<String, Boolean> tryParseInt = s -> {
            try {
                // Try parsing the string as int
                int asNum = Integer.parseInt(s);
                return true;
            } catch (NumberFormatException e) {
                return false;
            }
        };

        Boolean leftIsInt = tryParseInt.apply(left);
        Boolean rightIsInt = tryParseInt.apply(right);

        // Perform type-checking
        MethodHandle newmh;
        if (leftIsInt && rightIsInt) {
            newmh = lookup.findStatic(
                MyDynLangAdd.class,
                "addInts",
                MethodType.methodType(String.class, String.class, String.class)
            );
        } else if (leftIsInt && !rightIsInt) {
            throw new RuntimeException(
                "Cannot add String to Integer, LHS has to be String!"
            );
        } else {
            newmh = lookup.findStatic(
                MyDynLangAdd.class,
                "concatStrings",
                MethodType.methodType(String.class, String.class, String.class)
            );
        }

        // Insert automatic conversion from Object to String and vice versa
        newmh = newmh.asType(callsite.type());

        return (String) newmh.invoke(left, right);
    }
```

Now, every time after the first time we hit this *indy* call, we would directly call the dispatch function. This function can do the type-check, and then redirect the call to the correct function. This is exactly how languages like JRuby, Jython and Groovy use *indy* to dispatch calls at runtime.

As with any dynamically typed language, checking types at runtime incurs performance cost. Here are some ways we can further improve performance of *indy* calls:
- If we find that the types passed at a callsite are always the same, i.e., they are hardcoded constants, then we can ask the dispatcher to replace itself with the `addInts`/`concatStrings` functions at the CallSite.
- We can "Speculatively" store the methodHandle for one of the possible implementations at the callsite. When the speculation fails, perhaps via an exception, we can catch the exception and redirect to the dispatcher.

Though *indy* was introduced to support dynamic-typing in the JVM, Java also adapted it to make its MethodHandle and Reflection implementation better. But that's a story for another day!

*This post was written by a human and proof-read by an LLM*

### Footnotes
