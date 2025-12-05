# Code Models

#### Paul Sandoz {.author}

#### June 2024, Updated Dec 2025 {.date}

Code reflection, an enhancement to Java reflection, enables access to symbolic
representations of Java code in method and lambda expressions. "Symbolic
representations of Java code" may seem like a fancy term, but its easily
demystified. It's a model of Java code where the code of a method or lambda
expressions is represented as instances of specific Java classes arranged in an
appropriate structure. Thereby it is possible to write Java programs that
manipulate Java programs. Before we get into the details of how code reflection
models Java code we should talk about two existing approaches to modeling Java
code.

Java developers use a Java program every day that manipulates Java programs.
It's called the Java compiler. It has its own internal model of Java code,
instances of which are called Abstract Syntax Trees (ASTs). The AST model
closely aligns with the Java grammar as specified by the Java Language
Specification. The Java compiler parses source program text into instances of
specific Java classes that form the AST, traverses and manipulates the AST to
verify the source program is a correct, and if so generates class files
containing bytecode instructions.

Bytecode is another model of Java code, one that is standardized by the Java
Virtual Machine Specification and consumed by Java run times. The Java compiler
transforms Java code represented as instances of specific Java classes in the
AST model into Java code represented as specific classes in the bytecode model.
After which it can generate code attributes in class files. The OpenJDK HotSpot
runtime, a C++ program, manipulates the bytecode to interpret it or compile
parts of it to executable machine code.

The AST model and bytecode model naturally serve different purposes, and as a
result they have very different properties. However, the source program, the
AST, the bytecode, and even generated machine code, represent the same Java
program. The Java compiler and HotSpot runtime preserve Java program meaning
when they manipulate representations of Java code.

Neither the AST model nor the bytecode model is a suitable model for the
purposes of code reflection. We want to ensure code reflection's model of Java
code is broadly applicable across many use cases that may ingest Java code and
generate derived Java code or foreign code (such as differentiated Java code,
GPU code, or SQL statements). The AST model is too close to the source
containing too much information (syntactic details), and the bytecode model too
close to an executable form with useful information removed (types erased,
structures flattened). Both are hard to analyze and transform. So much so modern
compilers, in general, will commonly translate an AST-based model or
instruction-based (stack machine) model to a more appropriate model that is
easier to analyze and transform. Herein lies important clues as to the design of
code reflection's model.

## Code model design

Code reflection devises a third model of Java code, instances of which are
called _code models_. Code models for identified method and lambda expressions
are produced by the Java compiler, stored in class files, and made accessible at
run time via reflective APIs. The Java compiler will transform an AST to a Java
code model, in addition to generating bytecode instructions. Such code models
preserve Java program meaning. The code model will not contain all the syntactic
details as present in the AST but will retain type information and structural
information that is not present in bytecode. It is useful to think of code
models situated somewhere between ASTs and bytecode, initially closer to ASTs
than to the bytecode (further on we shall present how a code model may be
transformed and become closer to bytecode).

The code model design is heavily influenced by the design of data structures
used by many modern compilers to represent code. These data structures are
commonly referred to as Intermediate Representations (IRs). The design is
further influenced by Multi-Level Intermediate Representation (MLIR), a
sub-project of the LLVM Compiler Infrastructure project. Our intention is not to
compete with such compilers. We will focus on high-fidelity modeling of Java
code, manipulation of that code at a mid- to high-level, and interchange to
other models and compiler toolchains (native or otherwise). A particularly
challenging aspect of code reflection is ensuring the code model design and
respective Java API be broadly accessible to competent Java developers who don't
have PhDs in programming language theory and compilers (although we hope those
that do will enjoy using code reflection).

A code model contains code elements, operations, bodies, and blocks, that form a
tree. An operation contains zero or more bodies. A body contains one or more
blocks. A block contains a sequence of one or more operations. A block can
declare zero or more block parameters, values. An operation declares an
operation result, a value. An operation may use values as operands, but only
after they have been declared. A value has a type.

Code models are in Static Single Assignment (SSA) form, values can only be
assigned exactly once. The blocks within a body are interconnected with each
other and form a control flow graph. Values are also interconnected with each
other and form either expression graphs or use graphs. The relationship between
an operation result and its operands form part of an expression graph. The
relationship between a value and its uses form part of a use graph.

The Java API for code models has Java classes corresponding to operation, body,
block, block parameter, operation result, value, and value type. A code model
comprises instances of those Java classes arranged in the tree structure with
support for tree traversal. Additionally, a model supports graph traversal by
connecting blocks to blocks, operation results to operands, and values to
dependent values (uses).

Using this simple tree structure we can define specific operations, extending
from the Java class associated with an operation, that model many Java language
constructs, and therefore we can build code models that model many Java
programs. This may appear surprising at first. Readers may be more familiar with
term "operation" in a more conventional sense, such as arithmetic operations.
However, given the structure described above, there is no need to limit
ourselves to this conventional sense. We are free to define an operation whose
operational semantics model a method declaration, model a lambda expression, or
even model a `try` statement.

Code models are immutable. Code models can be produced by building, or by
transforming an existing code model. Transforming takes an input code model and
builds an output code model. For each input operation encountered in the input
code model we have the choice to add that operation to the builder of the output
code model (copying), to not add it (removal), or add new output operations
(replacement or addition).

This may all seem a little abstract so lets look at some examples.

> All the code presented in this article is available
> in [test source][test-source] located in the Babylon repository

[test-source]: https://github.com/openjdk/babylon/blob/code-reflection/test/jdk/java/lang/reflect/code/TestExpressionGraphs.java

## Code model access

Consider the following method, `sub`, that subtracts two values.

[//]: # (@formatter:off)
```java
@Reflect
static double sub(double a, double b) {
   return a - b;
}
```
[//]: # (@formatter:on)

We annotate it with `@Reflect` to identify that the method's code model
should be built by the compiler and made accessible at runtime using the
reflection API.

We find the `java.lang.reflect.Method` instance of `sub`, and then ask it for
its code model by invoking the method `Op.ofMethod`. Only methods annotated
with `@Reflect` will have code models, hence this method is partial.

(Note: since code reflection is an incubating feature we cannot add new APIs in
packages of other modules, such as in the `java.lang.reflect` package of the
`java.base` module. For now, we must provide such methods in the incubating code
reflection module.)

[//]: # (@formatter:off)
```java
// Get the reflective object for method sub
Method m = TestExpressionGraphs.class.getDeclaredMethod(
        "sub", double.class, double.class);
// Get the code model for method sub
Optional<CoreOp.FuncOp> oModel = Op.ofMethod(m);
CoreOp.FuncOp model = oModel.orElseThrow();
```
[//]: # (@formatter:on)

## Traversal of code model elements

`sub`'s code model is represented as an instance of `CoreOp.FuncOp`,
corresponding to a *function declaration* operation that models a Java method
declaration. What does the code model of `sub` look like? We can get a sense of
this by traversing the model, a tree, and printing out all the code elements.

[//]: # (@formatter:off)
```jshelllanguage
// Stream of elements topologically sorted in depth-first search pre-order
model.elements().forEach(codeElement -> {
    // Count the depth of the code element
    int depth = 0;
    CodeElement<?, ?> parent = codeElement;
    while ((parent = parent.parent()) != null) depth++;
    // Print out code element class
    System.out.println("  ".repeat(depth) + codeElement.getClass());
});
```
[//]: # (@formatter:on)

The lambda expression passed to the stream's `forEach` method is called for each
encountered _code element_ in the model, and it prints out the class name
prefixed with space proportional to the tree depth of the element. The output is
shown below.

```text
class jdk.incubator.code.dialect.core.CoreOp$FuncOp
  class jdk.incubator.code.Body
    class jdk.incubator.code.Block
      class jdk.incubator.code.dialect.core.CoreOp$VarOp
      class jdk.incubator.code.dialect.core.CoreOp$VarOp
      class jdk.incubator.code.dialect.core.CoreOp$VarAccessOp$VarLoadOp
      class jdk.incubator.code.dialect.core.CoreOp$VarAccessOp$VarLoadOp
      class jdk.incubator.code.dialect.java.JavaOp$SubOp
      class jdk.incubator.code.dialect.core.CoreOp$ReturnOp
```

We can observe that the top of the tree is the `CoreOp.FuncOp` which contains
one child, a `Body`, which in turn contains one child, a `Block`, which in turn
contains a sequence of operations.

The `CodeElement.elements` method returns a stream constructed with a gathering
operation that whose integrator recursively traverses code elements:

```java
default Stream<CodeElement<?, ?>> elements() {
    return Stream.of(this).gather(
            () -> (_, e, downstream) -> traversePreOrder(e, downstream));
}

private static boolean traversePreOrder(
        CodeElement<?, ?> e,
        Gatherer.Downstream<? super CodeElement<?, ?>> v) {
    if (!v.push(e)) {
        return false;
    }
    for (CodeElement<?, ?> c : e.children()) {
        if (!traversePreOrder(c, v)) {
            return false;
        }
    }
    return true;
}
```

So far we have seen that the code model API supports two kinds of tree traversal
of code elements:

1. up the tree, from child to parent when we calculated the depth of a code
   element; and
2. down the tree, from parent to children in the implementation of
   the `elements` method.

Later we shall explore traversal of values in code models.

## Explanation of code models by printing them

Our traversal that prints out the code element classes is not particularly
informative. A superior way to see what a code model looks like is to traverse
the model and print out more descriptive information about each code element.
Thankfully we don't need to write this ourselves. We can call the method
`toText` on the model that returns a string representation that we can then
print.

```jshelllanguage
System.out.println(model.toText());
```

Which outputs the following text.

```text
func @loc="46:5:file:/.../TestExpressionGraphs.java" @"sub"
(%0 : java.type:"double", %1 : java.type:"double")java.type:"double" -> {
    %2 : Var<java.type:"double"> = var %0 @loc="46:5" @"a";
    %3 : Var<java.type:"double"> = var %1 @loc="46:5" @"b";
    %4 : java.type:"double" = var.load %2 @loc="48:16";
    %5 : java.type:"double" = var.load %3 @loc="48:20";
    %6 : java.type:"double" = sub %4 %5 @loc="48:16";
    return %6 @loc="48:9";
};
```

> A code model's text is designed to be human-readable and is very useful for
> debugging models and for testing. It is also an invaluable for explaining
> code models, and we shall use it extensively in this article.

> To aid debugging each operation has line number information, and the root
> operation also has source information from where the code model originated.

The code model text shows the code model's root is a function declaration
(`func`) operation. The lambda-like expression represents the fusion of the
function declaration operation's single body and the body's first and only
block, called the entry block. Then there is a sequence of operations in the
entry block. For each operation there is an instance of a corresponding Java
class, all of which extend from the abstract class `jdk.incubator.code.Op`
and which have already seen when we printed out the classes. Unsurprisingly the
printed operations and printed operation classes occur in the same order since
the `toText` method traverses the model in the same order as we traversed.

> The function declaration operation has an operation result, like all other
> operations, but since it's the root of the tree and not used we don't
> present it.

The entry block has two block parameters, `%0` and `%1` each described by a type
of `java.type:"double"`, which model method `sub`'s initial values for
parameters `a` and `b`. The method parameters themselves (variables) are modeled
as `var` operations that are initialized with the corresponding block
parameters. The result of a `var` operation is value, a _variable value_, whose
type is a _variable type_, `Var<T>`. A variable value holds another value of
type `T`, the value of the variable, which can be loaded or stored using
variable access operations, respectively modeling an expression that denotes a
variable and assignment to a variable.

> Although `Var<T>` looks like a generic Java type it is not. Just as
> we can define a set of operations for use in code models we can also
> define a set of types. We have a set of operations for modeling Java code,
> and we also have a set of code model types for modeling Java types. In
> addition, we require some other non-Java types such as for the
> modeling of local variables or say the grouping of multiple values (tuples)
> where the rules for Java types do not apply.

The expressions denoting parameters `a` and `b` are modeled as `var.load`
operations. The results of these operations are _used_ as operands of other
operations. Likewise, subsequent operations also produce results, e.g., `%6` the
result of a subtraction operation, that is used as an operand of the
`return` operation.

> The `return` operation has a result, again like all other operations, but
> since that result cannot be meaningfully used we don't present it by default.

Now let us consider a slightly more complex method, `distance1`, that computes a
simple mathematical expression, the distance between two scalar values.

[//]: # (@formatter:off)
```java
@Reflect
static double distance1(double a, double b) {
   return Math.abs(a - b);
}
```
[//]: # (@formatter:on)

This is similar to the `sub` method except we now have an invocation
to `Math.abs` that operates on the result of the subtraction. How is that
invocation (a class invocation expression) represented in the code model? Or
alternatively how is the invocation expression modelled? To help answer this
question we can print out the code model, like we did previously.

```text
func @loc="51:5:file:/.../TestExpressionGraphs.java" @"distance1"
(%0 : java.type:"double", %1 : java.type:"double")java.type:"double" -> {
    %2 : Var<java.type:"double"> = var %0 @loc="51:5" @"a";
    %3 : Var<java.type:"double"> = var %1 @loc="51:5" @"b";
    %4 : java.type:"double" = var.load %2 @loc="53:25";
    %5 : java.type:"double" = var.load %3 @loc="53:29";
    %6 : java.type:"double" = sub %4 %5 @loc="53:25";
    %7 : java.type:"double" = invoke %6 @loc="53:16" @java.ref:"java.lang.Math::abs(double):double";
    return %7 @loc="53:9";
};
```

The invocation is modeled as an `invoke` operation, and its result is used as
the operand of the `return` operation. It accepts as an operand the result of
the `sub` operation. The `invoke` operation declares a method reference, a
symbolic description of the method `Math.abs`.

> In this case we know the `invoke` operation models a class invocation
> expression (a call to a static method) because the number of
> operands is the same as the number of described method parameters. An
> instance invocation expression would have a number of operands that is one
> more than the number described method parameters, where the first operand
> is the receiver.

## Code models and Static Single Assignment (SSA)

We can clearly see code models are in Static Single Assignment (SSA) form, and
there is no explicit distinction, as there is in the source code, between
statements and expressions. Block parameters and operation results are declared
before they are used and cannot be reassigned (and we therefore require special
operations and types to model variables). It's as if we were to rewrite
method `distance1` as say `distance1a`, where for each simple expression we
assign the result to a new final local variable and then subsequently use that
variable.

[//]: # (@formatter:off)
```java
@Reflect
static double distance1a(final double a, final double b) {
    final double diff = a - b;
    final double result = Math.abs(diff);
    return result;
}
```
[//]: # (@formatter:on)

> We are not encouraging developers to generally write code like this!
> Source code should be readable and maintainable. Code models have
> different requirements, and so the text of models is naturally not
> designed to be as readable and maintainable as the source code it
> originated from.

We can print out the code model for method `distance1a`.

```text
func @loc="56:5:file:///.../TestExpressionGraphs.java" @"distance1a"
(%0 : java.type:"double", %1 : java.type:"double")java.type:"double" -> {
    %2 : Var<java.type:"double"> = var %0 @loc="56:5" @"a";
    %3 : Var<java.type:"double"> = var %1 @loc="56:5" @"b";
    %4 : java.type:"double" = var.load %2 @loc="58:29";
    %5 : java.type:"double" = var.load %3 @loc="58:33";
    %6 : java.type:"double" = sub %4 %5 @loc="58:29";
    %7 : Var<java.type:"double"> = var %6 @loc="58:9" @"diff";
    %8 : java.type:"double" = var.load %7 @loc="59:40";
    %9 : java.type:"double" = invoke %8 @loc="59:31" @java.ref:"java.lang.Math::abs(double):double";
    %10 : Var<java.type:"double"> = var %9 @loc="59:9" @"result";
    %11 : java.type:"double" = var.load %10 @loc="60:16";
    return %11 @loc="60:9";
};
```

The model looks very similar to `distance1`'s model, except that we now have
additional `var` operations modeling _local_ variables `diff` and `result`. Even
though there are differences both methods and their models are equivalent in
terms of program behaviour (ignoring the effects of debugging). We can show this
by performing a pure SSA transformation on both models and comparing them.

```java
CoreOp.FuncOp ssaModel = SSA.transform(model);
```

Such a transformation removes the variable operations, replacing the use of
their results with their operands.

Here are the two models after transforming.

```text
func @loc="51:5:file:///.../TestExpressionGraphs.java" @"distance1"
(%0 : java.type:"double", %1 : java.type:"double")java.type:"double" -> {
    %2 : java.type:"double" = sub %0 %1 @loc="53:25";
    %3 : java.type:"double" = invoke %2 @loc="53:16" @java.ref:"java.lang.Math::abs(double):double";
    return %3 @loc="53:9";
};

func @loc="56:5:file:///.../TestExpressionGraphs.java" @"distance1a"
(%0 : java.type:"double", %1 : java.type:"double")java.type:"double" -> {
    %2 : java.type:"double" = sub %0 %1 @loc="58:29";
    %3 : java.type:"double" = invoke %2 @loc="59:31" @java.ref:"java.lang.Math::abs(double):double";
    return %3 @loc="60:9";
};
```

Apart from the difference in location details the two models are identical, and
have become easier to analyse for certain use cases.

> Notice the SSA transformation has preserved location information on
> operations that were copied

## Code models with simple control flow

Let's further modify `distance1b` by replacing the method invocation to
`Math.abs` with an (almost) equivalent inlined expression using the conditional
operator `? :`.

[//]: # (@formatter:off)
```java
@Reflect
static double distance1b(final double a, final double b) {
    final double diff = a - b;
    // Note, incorrect for negative zero values
    final double result = diff < 0d ? -diff : diff;
    return result;
}
```
[//]: # (@formatter:on)

We now have some control flow in the expression whose result is assigned to
local variable `result`. How do we model conditional operator `? :`? To find out
let's print out `distance1b`'s model.

```text
func @loc="63:5:file:///.../TestExpressionGraphs.java" @"distance1b"
(%0 : java.type:"double", %1 : java.type:"double")java.type:"double" -> {
    %2 : Var<java.type:"double"> = var %0 @loc="63:5" @"a";
    %3 : Var<java.type:"double"> = var %1 @loc="63:5" @"b";
    %4 : java.type:"double" = var.load %2 @loc="65:29";
    %5 : java.type:"double" = var.load %3 @loc="65:33";
    %6 : java.type:"double" = sub %4 %5 @loc="65:29";
    %7 : Var<java.type:"double"> = var %6 @loc="65:9" @"diff";
    %8 : java.type:"double" = java.cexpression @loc="67:31"
        ()java.type:"boolean" -> {
            %9 : java.type:"double" = var.load %7 @loc="67:31";
            %10 : java.type:"double" = constant @loc="67:38" @0.0d;
            %11 : java.type:"boolean" = lt %9 %10 @loc="67:31";
            yield %11 @loc="67:31";
        }
        ()java.type:"double" -> {
            %12 : java.type:"double" = var.load %7 @loc="67:44";
            %13 : java.type:"double" = neg %12 @loc="67:43";
            yield %13 @loc="67:31";
        }
        ()java.type:"double" -> {
            %14 : java.type:"double" = var.load %7 @loc="67:51";
            yield %14 @loc="67:31";
        };
    %15 : Var<java.type:"double"> = var %8 @loc="67:9" @"result";
    %16 : java.type:"double" = var.load %15 @loc="68:16";
    return %16 @loc="68:9";
};
```

The `java.cexpression` operation models the conditional operator `? :`. It
contains three bodies, each with one block. Each expression of the conditional
operator `? :` is modeled as a body, and therefore we capture code structure
associated with control flow. The operation specifies how control flows between
its bodies according to Java program behaviour as specified by the Java Language
Specification. Many operations modeling more complex Java language expressions
and statements will follow a similar pattern.

Sometimes it's useful to process a model with a `java.cexpression` operation but
in other cases it may be problematic as we need to understand the operation's
behaviour. It is possible to replace a `java.cexpression` operation with other
code elements that explicitly model the operation's control flow in a more basic
and general form. We can perform such replacement with a _transformation_
that _lowers_ such operations.

[//]: # (@formatter:off)
```jshelllanguage
CoreOp.FuncOp loweredModel = model.transform(OpTransformer.LOWERING_TRANSFORMER);
```
[//]: # (@formatter:on)

The `transform` method traverses a model and builds a new model. It accepts a
transformer function as an argument that implements the transformation. In this
case the transformer `LOWERING_TRANSFORMER` lowers operations that are capable
of being lowered, such as the `java.cexpression` operation (there are many other
lowerable operations, such as the operation modelling a `for` loop that we shall
see later.)

Printing out the lowered model reveals the replacing code elements.

```text
func @loc="63:5:file:///.../TestExpressionGraphs.java" @"distance1b"
(%0 : java.type:"double", %1 : java.type:"double")java.type:"double" -> {
    %2 : Var<java.type:"double"> = var %0 @loc="63:5" @"a";
    %3 : Var<java.type:"double"> = var %1 @loc="63:5" @"b";
    %4 : java.type:"double" = var.load %2 @loc="65:29";
    %5 : java.type:"double" = var.load %3 @loc="65:33";
    %6 : java.type:"double" = sub %4 %5 @loc="65:29";
    %7 : Var<java.type:"double"> = var %6 @loc="65:9" @"diff";
    %8 : java.type:"double" = var.load %7 @loc="67:31";
    %9 : java.type:"double" = constant @loc="67:38" @0.0d;
    %10 : java.type:"boolean" = lt %8 %9 @loc="67:31";
    cbranch %10 ^block_1 ^block_2;
  
  ^block_1:
    %11 : java.type:"double" = var.load %7 @loc="67:44";
    %12 : java.type:"double" = neg %11 @loc="67:43";
    branch ^block_3(%12);
  
  ^block_2:
    %13 : java.type:"double" = var.load %7 @loc="67:51";
    branch ^block_3(%13);
  
  ^block_3(%14 : java.type:"double"):
    %15 : Var<java.type:"double"> = var %14 @loc="67:9" @"result";
    %16 : java.type:"double" = var.load %15 @loc="68:16";
    return %16 @loc="68:9";
};
```

We can clearly see three new blocks have been added to the `func`
operation's body, `^block_1`, `^block_2`, `^block_3`, and they are
interconnected. They form a _control-flow graph_.

The `func` operation body's entry block contains the same operations in the
prior model up to the `java.cexpression` operation. Then all operations, except
the last, in the first body of the `java.cexpression` operation have been
appended to the entry block. All operations, except the last, in the second body
of the `java.cexpression` operation have been appended to `^block_1`. All
operations, except the last, in the third body of the `java.cexpression`
operation have been appended to `^block_2`. Finally `^block_3` contains the same
operations in the prior model that occur after the `java.cexpression` operation.

The last operation in each body of the `java.cexpression` operation,
`yield` operations, are replaced with a branch operations. The entry block
branches conditionally to a _successor_ block, either `^block_1` or `^block_2`
based on its boolean operand. Both of those blocks branch unconditionally to
successor `^block_3`, and they each pass their yielded result as a block
argument. `^block_3` has a block parameter, `%14`, that replaces the result of
the `java.cexpression` operation.

> Block parameter, `%14`, represents a value that comes from
> two control flow paths. Such values are equivalent to PHI (Φ) nodes or
> PHI instructions in other intermediate representations. Block arguments and
> block parameters look and feel like function arguments and function
> parameters.
> Blocks look like functions. And, branches to blocks look like function
> calls or tail calls. This is much easier for developers to understand than PHI
> nodes.

We can observe that the child blocks of a body occur in a specific order,
reverse postorder where generally a block occurs before its successor(s). This
order is useful for control-flow analysis.

> Reverse postorder is a topological sort of the blocks in the control-flow
> graph

Transforming the lowered model with the SSA transformation (presented earlier)
results in a simpler model.

```text
func @loc="63:5:file:///.../TestExpressionGraphs.java" @"distance1b"
(%0 : java.type:"double", %1 : java.type:"double")java.type:"double" -> {
    %2 : java.type:"double" = sub %0 %1 @loc="65:29";
    %3 : java.type:"double" = constant @loc="67:38" @0.0d;
    %4 : java.type:"boolean" = lt %2 %3 @loc="67:31";
    cbranch %4 ^block_1 ^block_2;
  
  ^block_1:
    %5 : java.type:"double" = neg %2 @loc="67:43";
    branch ^block_3(%5);
  
  ^block_2:
    branch ^block_3(%2);
  
  ^block_3(%6 : java.type:"double"):
    return %6 @loc="68:9";
};
```

> The SSA transformation is implemented as a code model transformer, like
> the lowering transformer, it's just implemented in the `SSA.transform` method

All three models have the same program behaviour as the original Java program.

## Code models with more complex control flow

Let's enhance the distance function to compute the distance between two
N-dimensional points.

[//]: # (@formatter:off)
```java
@Reflect
static double distanceN(double[] a, double[] b) {
    double sum = 0d;
    for (int i = 0; i < a.length; i++) {
        sum += Math.pow(a[i] - b[i], 2d);
    }
    return Math.sqrt(sum);
}
```
[//]: # (@formatter:on)

We loop over the number of dimensions, sum the square of the distance between
each dimension, and then return the square root of the final sum.

How do we model the `for` statement? To find out let's print out `distanceN`'s
model.

```text
func @loc="71:5:file:///.../TestExpressionGraphs.java" @"distanceN"
(%0 : java.type:"double[]", %1 : java.type:"double[]")java.type:"double" -> {
    %2 : Var<java.type:"double[]"> = var %0 @loc="71:5" @"a";
    %3 : Var<java.type:"double[]"> = var %1 @loc="71:5" @"b";
    %4 : java.type:"double" = constant @loc="73:22" @0.0d;
    %5 : Var<java.type:"double"> = var %4 @loc="73:9" @"sum";
    java.for @loc="74:9"
        ()Var<java.type:"int"> -> {
            %6 : java.type:"int" = constant @loc="74:22" @0;
            %7 : Var<java.type:"int"> = var %6 @loc="74:14" @"i";
            yield %7 @loc="74:9";
        }
        (%8 : Var<java.type:"int">)java.type:"boolean" -> {
            %9 : java.type:"int" = var.load %8 @loc="74:25";
            %10 : java.type:"double[]" = var.load %2 @loc="74:29";
            %11 : java.type:"int" = array.length %10 @loc="74:29";
            %12 : java.type:"boolean" = lt %9 %11 @loc="74:25";
            yield %12 @loc="74:9";
        }
        (%13 : Var<java.type:"int">)java.type:"void" -> {
            %14 : java.type:"int" = var.load %13 @loc="74:39";
            %15 : java.type:"int" = constant @loc="74:39" @1;
            %16 : java.type:"int" = add %14 %15 @loc="74:39";
            var.store %13 %16 @loc="74:39";
            yield @loc="74:9";
        }
        (%17 : Var<java.type:"int">)java.type:"void" -> {
            %18 : java.type:"double" = var.load %5 @loc="75:13";
            %19 : java.type:"double[]" = var.load %2 @loc="75:29";
            %20 : java.type:"int" = var.load %17 @loc="75:31";
            %21 : java.type:"double" = array.load %19 %20 @loc="75:29";
            %22 : java.type:"double[]" = var.load %3 @loc="75:36";
            %23 : java.type:"int" = var.load %17 @loc="75:38";
            %24 : java.type:"double" = array.load %22 %23 @loc="75:36";
            %25 : java.type:"double" = sub %21 %24 @loc="75:29";
            %26 : java.type:"double" = constant @loc="75:42" @2.0d;
            %27 : java.type:"double" = invoke %25 %26 @loc="75:20" @java.ref:"java.lang.Math::pow(double, double):double";
            %28 : java.type:"double" = add %18 %27 @loc="75:13";
            var.store %5 %28 @loc="75:13";
            java.continue @loc="74:9";
        };
    %29 : java.type:"double" = var.load %5 @loc="77:26";
    %30 : java.type:"double" = invoke %29 @loc="77:16" @java.ref:"java.lang.Math::sqrt(double):double";
    return %30 @loc="77:9";
};
```

The `java.for` operation models the `for` statement. There are four bodies
corresponding, in order, to four nonterminal symbols in the grammar specified in
the Java Language Specification:

```text
BasicForStatement:
  for ( [ForInit] ; [Expression] ; [ForUpdate] ) Statement 
```

which also states (in section [14.14.1][jls-14.14.1]):

> The basic for statement executes some initialization code, then executes an
> _Expression_, a _Statement_, and some update code repeatedly until the
> value of the _Expression_ is false.

[jls-14.14.1]: https://docs.oracle.com/javase/specs/jls/se25/html/jls-14.html#jls-14.14.1

We can see that the first body corresponds to the initialization code. It yields
a variable value modeling local variable `i`. This variable value then _flows_
as a parameter to all the other bodies and therefore they can access `i`.

> In general a for loop can declare zero or more local variables and
> therefore the first body may yield zero or more variable values. Two or more
> variable values are wrapped in a yielded tuple value, since code
> models do not intrinsically support the grouping of multiple return values or
> an operation producing multiple results.

The second body corresponds to the _Expression_ that models the code checking
whether local variable `i` is less than the array length. The third body
corresponds to the update code that increments `i`. And, the fourth body
corresponds to the _Statement_ that performs the intermediate computation for
each dimension.

Like with the code model for `distance1b` we can replace the `java.for`
operation with other code elements by performing the same lowering
transformation. Printing out the lowered model reveals the replacing code
elements.

```text
func @loc="71:5:file:///.../TestExpressionGraphs.java" @"distanceN"
(%0 : java.type:"double[]", %1 : java.type:"double[]")java.type:"double" -> {
    %2 : Var<java.type:"double[]"> = var %0 @loc="71:5" @"a";
    %3 : Var<java.type:"double[]"> = var %1 @loc="71:5" @"b";
    %4 : java.type:"double" = constant @loc="73:22" @0.0d;
    %5 : Var<java.type:"double"> = var %4 @loc="73:9" @"sum";
    %6 : java.type:"int" = constant @loc="74:22" @0;
    %7 : Var<java.type:"int"> = var %6 @loc="74:14" @"i";
    branch ^block_1;
  
  ^block_1:
    %8 : java.type:"int" = var.load %7 @loc="74:25";
    %9 : java.type:"double[]" = var.load %2 @loc="74:29";
    %10 : java.type:"int" = array.length %9 @loc="74:29";
    %11 : java.type:"boolean" = lt %8 %10 @loc="74:25";
    cbranch %11 ^block_2 ^block_4;
  
  ^block_2:
    %12 : java.type:"double" = var.load %5 @loc="75:13";
    %13 : java.type:"double[]" = var.load %2 @loc="75:29";
    %14 : java.type:"int" = var.load %7 @loc="75:31";
    %15 : java.type:"double" = array.load %13 %14 @loc="75:29";
    %16 : java.type:"double[]" = var.load %3 @loc="75:36";
    %17 : java.type:"int" = var.load %7 @loc="75:38";
    %18 : java.type:"double" = array.load %16 %17 @loc="75:36";
    %19 : java.type:"double" = sub %15 %18 @loc="75:29";
    %20 : java.type:"double" = constant @loc="75:42" @2.0d;
    %21 : java.type:"double" = invoke %19 %20 @loc="75:20" @java.ref:"java.lang.Math::pow(double, double):double";
    %22 : java.type:"double" = add %12 %21 @loc="75:13";
    var.store %5 %22 @loc="75:13";
    branch ^block_3;
  
  ^block_3:
    %23 : java.type:"int" = var.load %7 @loc="74:39";
    %24 : java.type:"int" = constant @loc="74:39" @1;
    %25 : java.type:"int" = add %23 %24 @loc="74:39";
    var.store %7 %25 @loc="74:39";
    branch ^block_1;
  
  ^block_4:
    %26 : java.type:"double" = var.load %5 @loc="77:26";
    %27 : java.type:"double" = invoke %26 @loc="77:16" @java.ref:"java.lang.Math::sqrt(double):double";
    return %27 @loc="77:9";
};
```

> Both the `java.for` operation and the `java.cexpression` operation
> implement their own replacement. The corresponding operation classes
> extend from `Op.Lowerable` interface, which declares an abstract method,
> `lower`, that each operation implements.

The `func` operation's body contains a control flow graph. Notice that
`^block_3` branches to `^block_1`, which is commonly referred to as a _back
branch_. This models continuation of the loop.

Transforming the lowered model with the SSA transformation again results in a
simpler model.

```text
func @loc="71:5:file:///.../TestExpressionGraphs.java" @"distanceN"
(%0 : java.type:"double[]", %1 : java.type:"double[]")java.type:"double" -> {
    %2 : java.type:"double" = constant @loc="73:22" @0.0d;
    %3 : java.type:"int" = constant @loc="74:22" @0;
    branch ^block_1(%3, %2);
  
  ^block_1(%4 : java.type:"int", %5 : java.type:"double"):
    %6 : java.type:"int" = array.length %0 @loc="74:29";
    %7 : java.type:"boolean" = lt %4 %6 @loc="74:25";
    cbranch %7 ^block_2 ^block_4;
  
  ^block_2:
    %8 : java.type:"double" = array.load %0 %4 @loc="75:29";
    %9 : java.type:"double" = array.load %1 %4 @loc="75:36";
    %10 : java.type:"double" = sub %8 %9 @loc="75:29";
    %11 : java.type:"double" = constant @loc="75:42" @2.0d;
    %12 : java.type:"double" = invoke %10 %11 @loc="75:20" @java.ref:"java.lang.Math::pow(double, double):double";
    %13 : java.type:"double" = add %5 %12 @loc="75:13";
    branch ^block_3;
  
  ^block_3:
    %14 : java.type:"int" = constant @loc="74:39" @1;
    %15 : java.type:"int" = add %4 %14 @loc="74:39";
    branch ^block_1(%15, %13);
  
  ^block_4:
    %16 : java.type:"double" = invoke %5 @loc="77:16" @java.ref:"java.lang.Math::sqrt(double):double";
    return %16 @loc="77:9";
};
```

`^block_1` now has two block parameters, `%4` and `%5`, corresponding to the
values of local variables `i` and `sum` respectively for the current loop
iteration. The (back) branch in `^block_3` passes the values to be used for the
next loop iteration as block arguments.

> A value can be used by an operation if it is defined earlier in the same
> block or defined in a dominating block. This is why the `add` operation
> in `^block_2` or the `invoke` operation in `^block_4` can use `%5`, since
> `^block_1` dominates `^block_2` and `^block_4`.

> Operations with structured control flow and pure SSA form are not mutually
> exclusive. Although we will not model Java expressions and statement
> with control flow in such a manner the code model design itself does not have
> such limitations (see the [Triton example][Triton-example] for using code
> models to model non-Java programs).

[Triton-example]: https://openjdk.org/projects/babylon/articles/triton

## Expression graphs and use graphs

So far we have shown tree traversal of a code model's elements (operations,
bodies, and blocks). There are other ways to traverse _items_ of a code model,
specifically the traversal of values. Given an operation result we can traverse
to the values that are the operation's operands, and so on, to produce an
expression graph. We can also think about the reverse. Given a value we can
traverse to the operation results of operations that _uses_ it as an operand,
and so on, to produce a use graph.

> It is an expression _graph_ because two or more operations may use the
> same value as an operand. An expression graph cannot have cycles, so it is
> also acyclic. Conceptually an expression graph traverses up the code model.
>
> It is a use _graph_ because a value can be used by more two or more operations
> whose results are all subsequently used by another operation. A use graph
> is also acyclic. Conceptually a use graph traverses down the code model.

In this section we shall show how to traverse expression graphs and use graphs
and build up simple graph structures. We declare a record class that represents
a node in a graph. A node has two components, a value associated with the node
and a list of outgoing edges to other nodes.

```java
record Node<T>(T value, List<Node<T>> edges) {
}
```

Then, we implement a method, `expressionGraph`, that computes the expression
graph for a given value.

```java
static Node<Value> expressionGraph(Value value) {
    return expressionGraph(new HashMap<>(), value);
}

static Node<Value> expressionGraph(Map<Value, Node<Value>> visited, Value value) {
    // If value has already been visited return its node
    if (visited.containsKey(value)) {
        return visited.get(value);
    }

    // Find the expression graphs for each operand
    List<Node<Value>> edges = new ArrayList<>();
    for (Value operand : value.dependsOn()) {
        edges.add(expressionGraph(operand));
    }
    Node<Value> node = new Node<>(value, edges);
    visited.put(value, node);
    return node;
}
```

Given a value the corresponding node's edges are the nodes produced by
recursively computing the expression graphs for the set of values the value
_depends on_. If the value is an operation result then that set will be the set
of the operation's operands. It is a set because an operation may use a value
two or more times as two or more operands. If the value is a block parameter it
depends on no other values so the set is empty. Since we are creating a graph we
also need to check if we have already visited a value, if so we reuse its
corresponding node.

It is instructive to show an alternative implementation of`expressionGraph`
that performs explicit instance of checks on the value. However, in general it
is recommended the method `Value.dependsOn` be used instead.

```java
static Node<Value> expressionGraph(Map<Value, Node<Value>> visited, Value value) {
    // If value has already been visited return its node
    if (visited.containsKey(value)) {
        return visited.get(value);
    }

    List<Node<Value>> edges;
    if (value instanceof Op.Result result) {
        edges = new ArrayList<>();
        // Find the expression graphs for each operand
        Set<Value> valueVisited = new HashSet<>();
        for (Value operand : result.op().operands()) {
            // Ensure an operand is visited only once
            if (valueVisited.add(operand)) {
                edges.add(expressionGraph(operand));
            }
        }
        // TODO if terminating operation find expression graphs
        //      for each successor argument
    } else {
        assert value instanceof Block.Parameter;
        // A block parameter has no outgoing edges
        edges = List.of();
    }
    Node<Value> node = new Node<>(value, edges);
    visited.put(value, node);
    return node;
}
```

Given a value we test if the value is an instance of an operation result. If so,
the corresponding node's edges are the nodes produced by recursively computing
the expression graphs for the operation's operands. Otherwise, a value is an
instance of a block parameter, and we create a node with no edges.

> The TODO comment indicates that an operation result depends on the
> operation's operands and also its successor arguments, if the operation is
> the terminating (or last) operation in a block. Method `Op.Result.dependsOn`
> will return the set of operands and successor arguments.
> We have already seen such operations, branch operations, when looking at the
> lowered code models of methods `distance1b` and `distanceN`.

Finally, we implement a method, `useGraph`, that computes the use graph for a
given value.

```java
static Node<Value> useGraph(Value value) {
    return useGraph(new HashMap<>(), value);
}

static Node<Value> useGraph(Map<Value, Node<Value>> visited, Value value) {
    // If value has already been visited return its node
    if (visited.containsKey(value)) {
        return visited.get(value);
    }

    // Find the use graphs for each use
    List<Node<Value>> edges = new ArrayList<>();
    for (Op.Result use : value.uses()) {
        edges.add(useGraph(visited, use));
    }
    Node<Value> node = new Node<>(value, edges);
    visited.put(value, node);
    return node;
}
```

The method `useGraph` is similarly structured to method `expressionGraph`,
except that the corresponding node's edges are the nodes produced by recursively
computing the use graphs for the values uses.

Now we can start producing graphs for some of the models we have previously
presented. Let's take another look at the `distance1`. What does the expression
graph look like for the `return` operation? Let's compute the graph and print it
out along with the code model for comparison.

[//]: # (@formatter:off)
```java
CoreOp.FuncOp model = ...;
// Create the expression graph for the terminating operation result
Op.Result returnResult = model.body().entryBlock().terminatingOp().result();
Node<Value> returnGraph = expressionGraph(returnResult);
// Transform from Node<Value> to Node<String> and print the graph
System.out.println(returnGraph.transformValues(v -> printValue(names, v)));
```
[//]: # (@formatter:on)

```text
@Reflect
static double distance1(double a, double b) {
   return Math.abs(a - b);
}

func @loc="51:5:file:///.../TestExpressionGraphs.java"
@"distance1" (%0 : java.type:"double", %1 : java.type:"double")java.type:"double" -> {
    %2 : Var<java.type:"double"> = var %0 @loc="51:5" @"a";
    %3 : Var<java.type:"double"> = var %1 @loc="51:5" @"b";
    %4 : java.type:"double" = var.load %2 @loc="53:25";
    %5 : java.type:"double" = var.load %3 @loc="53:29";
    %6 : java.type:"double" = sub %4 %5 @loc="53:25";
    %7 : java.type:"double" = invoke %6 @loc="53:16" @java.ref:"java.lang.Math::abs(double):double";
    %8 : java.type:"void" = return %7 @loc="53:9";
};

%8 : java.type:"void" = return %7 @loc="53:9";
└── %7 : java.type:"double" = invoke %6 @loc="53:16" @java.ref:"java.lang.Math::abs(double):double";
    └── %6 : java.type:"double" = sub %4 %5 @loc="53:25";
        ├── %4 : java.type:"double" = var.load %2 @loc="53:25";
        │   └── %2 : Var<java.type:"double"> = var %0 @loc="51:5" @"a";
        │       └── %0 <block parameter>
        └── %5 : java.type:"double" = var.load %3 @loc="53:29";
            └── %3 : Var<java.type:"double"> = var %1 @loc="51:5" @"b";
                └── %1 <block parameter>
```

> Note that we are printing out the graph as a tree, so if a node has two or
> more outgoing edges it would be printed out two or more times, which is  
> fine because there are no cycles. In this case the expression graph is a
> tree, each non-root node has one incoming edge.

We can see that the expression graph bears a striking resemblance to an
_abstract syntax tree_. As we can see such trees are present in code models even
if initially it may not be so obvious that they are. Reversing the lines of the
printed expression tree, and removing the lines associated with the block
parameters, reveals a similar order to the operations in the function
declaration operation.

What do the use graphs look like for each of the function's parameters?
Similarly, lets compute the two use graphs and print them out.

[//]: # (@formatter:off)
```java
for (Block.Parameter parameter : model.parameters()) {
   Node<Value> useNode = useGraph(parameter);
   System.out.println(useNode.transformValues(v -> printValue(names, v)));
}
```
[//]: # (@formatter:on)

```text
@Reflect
static double distance1(double a, double b) {
   return Math.abs(a - b);
}

%0 <block parameter>
└── %2 : Var<java.type:"double"> = var %0 @loc="51:5" @"a";
    └── %4 : java.type:"double" = var.load %2 @loc="53:25";
        └── %6 : java.type:"double" = sub %4 %5 @loc="53:25";
            └── %7 : java.type:"double" = invoke %6 @loc="53:16" @java.ref:"java.lang.Math::abs(double):double";
                └── %8 : java.type:"void" = return %7 @loc="53:9";

%1 <block parameter>
└── %3 : Var<java.type:"double"> = var %1 @loc="51:5" @"b";
    └── %5 : java.type:"double" = var.load %3 @loc="53:29";
        └── %6 : java.type:"double" = sub %4 %5 @loc="53:25";
            └── %7 : java.type:"double" = invoke %6 @loc="53:16" @java.ref:"java.lang.Math::abs(double):double";
                └── %8 : java.type:"void" = return %7 @loc="53:9";
```

The use graph follows the same order as the operations in the function
declaration operation. We can also observe that the use graphs are sub-graphs of
the expression graph (in this case they are reverse paths).

What if we want to compute the expression graphs for all values in a code model?
One approach would be to apply the `expressionGraph` method to each and every
value. Alternatively, we can compute all the expression graphs by traversing the
code mode elements, in a similar manner to how we printed code element classes.

```java
static Map<Value, Node<Value>> expressionGraphs(CoreOp.FuncOp f) {
    return expressionGraphs(f.body());
}

static Map<Value, Node<Value>> expressionGraphs(Body b) {
    LinkedHashMap<Value, Node<Value>> graphs = new LinkedHashMap<>();
    b.elements().forEach(codeElement -> {
        switch (codeElement) {
            case Body _ -> {
                // Do nothing
            }
            case Block block -> {
                // Create the expression graphs for each block parameter
                // A block parameter has no outgoing edges
                for (Block.Parameter parameter : block.parameters()) {
                    graphs.put(parameter, new Node<>(parameter, List.of()));
                }
            }
            case Op op -> {
                // Find the expression graphs for each operand
                List<Node<Value>> edges = new ArrayList<>();
                for (Value operand : op.result().dependsOn()) {
                    // Get expression graph for the operand
                    // It must be previously computed since we encounter the
                    // declaration of values before their use
                    edges.add(graphs.get(operand));
                }
                // Create the expression graph for this operation result
                graphs.put(op.result(), new Node<>(op.result(), edges));
            }
        }
    });
    return graphs;
}
```

> The switch statement is exhaustive and does not require a default clause.
> `Body`, `Block`, and `Op` extend from the sealed abstract class
> `CodeElement` which permits only those prior classes.

This approach works because code elements are traversed in a specific order,
where values are declared before they are used. Therefore, we don't need to
track visited values as before. The node for an operand is guaranteed to be
present in `graphs`, the map of value to node. This approach also happens to be
more efficient than directly producing the expression graphs for each value,
since we can share node instances.

> Since code models are immutable the computed expressions graphs are stable
> and can never become out-of-sync with the model they are associated with.

We can easily verify both methods produce the same graphs by comparing the
`return` operation's expression graph computed by each method.

[//]: # (@formatter:off)
```java
// Create the expression graphs for all values
Map<Value, Node<Value>> graphs = expressionGraphs(model);
// The graphs for the terminating operation result are the same
assert returnGraph.equals(graphs.get(returnGraph.value()));
```
[//]: # (@formatter:on)

> We rely on record's capability to automatically implement the `equals`
> method.

## Root expression graphs and trees

Now that we know how to produce expression graphs we can start categorizing and
manipulating graphs based on certain rules that, for example, identify parts of
a code model that model statements or expressions. Why would we want to do this?
Apart from presenting further details on how to analyse code models for
analysis’ sake this does have practical use. Specifically, for the translation
for code models to C source code. Two use cases come to mind.

The [Babylon GPU work][babylon-hat] requires the transformation of code models
to GPU kernels (methods that execute on GPU hardware). One approach is to
transform code models to OpenCL C source or CUDA C source and compile using the
GPU-specific toolchains. We would like the transformed source to be idiomatic
(approximately as if a written by hand), allowing for easier debugging and
enabling the compilers to better optimize (since they likely better optimize
idiomatic code). Identifying expression graphs that model statements is useful
for the generation of idiomatic C code.

[babylon-hat]: https://github.com/openjdk/babylon/tree/code-reflection/hat

> Note the Babylon GPU work is also exploring the transformation of code
> models to kernels consisting of SPIRV or PTX instructions. Thereby
> we will thoroughly explore many options and help ensure code reflection is
> fit for purpose.

The Foreign Function and Memory API (Project Panama) supports the binding of a
Java method to a native function pointer so that the Java method can be invoked
natively via that function pointer. Such invocation is commonly referred to as
an upcall. Panama's upcalls are very efficient, but there is still a cost
transitioning from native to Java and back again. This transition can be removed
if we can access the code model of the Java method, translate it to C code,
compile it to native code, and bind the function pointer to that native code. It
would likely be applicable only to Java methods with simple expressions and
statements, e.g, methods whose behaviour is a function of their input. Again,
identifying expressions graphs that model statements is useful to determine
whether the Java method is applicable for transformation and for the generation
of idiomatic C code.

With those two use cases in mind lets focus on further analyzing expression
Given all the expressions graphs we can filter them, selecting graphs that are
considered _roots_. Let's initially define a _root expression graph_ as a graph
whose root node value has no uses. Then we can filter the graphs as follows.

[//]: # (@formatter:off)
```java
// Filter for root graphs, operation results with no uses
List<Node<Value>> rootGraphs = graphs.values().stream()
        .filter(n -> n.value() instanceof Op.Result opr &&
                switch (opr.op()) {
                    // An operation result with no uses
                    default -> opr.uses().isEmpty();
                })
        .toList();
```
[//]: # (@formatter:on)

> We will add another case to the switch statement later on

For the purposes of this section we shall focus on another method,
`squareDiff`, that computes the difference between two squares.

[//]: # (@formatter:off)
```java
@Reflect
static double squareDiff(double a, double b) {
    // a^2 - b^2 = (a + b) * (a - b)
    final double plus = a + b;
    final double minus = a - b;
    return plus * minus;
}
```
[//]: # (@formatter:on)

We similarly structure the method like `distance1a`, with multiple variable
declaration statements. In addition, parameters `a` and `b` are used more than
once. These features will exercise the analysis. The `squareDiff` method
contains one such root expression graph associated with the `return` operation.
Here it is.

```text
%15 : java.type:"void" = return %14 @loc="85:9";
└── %14 : java.type:"double" = mul %12 %13 @loc="85:16";
    ├── %12 : java.type:"double" = var.load %7 @loc="85:16";
    │   └── %7 : Var<java.type:"double"> = var %6 @loc="83:9" @"plus";
    │       └── %6 : java.type:"double" = add %4 %5 @loc="83:29";
    │           ├── %4 : java.type:"double" = var.load %2 @loc="83:29";
    │           │   └── %2 : Var<java.type:"double"> = var %0 @loc="80:5" @"a";
    │           │       └── %0 <block parameter>
    │           └── %5 : java.type:"double" = var.load %3 @loc="83:33";
    │               └── %3 : Var<java.type:"double"> = var %1 @loc="80:5" @"b";
    │                   └── %1 <block parameter>
    └── %13 : java.type:"double" = var.load %11 @loc="85:23";
        └── %11 : Var<java.type:"double"> = var %10 @loc="84:9" @"minus";
            └── %10 : java.type:"double" = sub %8 %9 @loc="84:30";
                ├── %8 : java.type:"double" = var.load %2 @loc="84:30";
                │   └── %2 : Var<java.type:"double"> = var %0 @loc="80:5" @"a";
                │       └── %0 <block parameter>
                └── %9 : java.type:"double" = var.load %3 @loc="84:34";
                    └── %3 : Var<java.type:"double"> = var %1 @loc="80:5" @"b";
                        └── %1 <block parameter>
```

Notice that the method has three statements, the two variable declaration
statements and the return statement, and yet there is only one root expression
graph.

Also notice that the graph contains the results of variable declaration
operations, modeling the local variable declaration statements (values `%7`
and `%11` for local variables `plus` and `minus` respectively), and also those
modeling the method parameter declarations (values `%2` and `%3` for
parameters `a` and `b` respectively). The results are used by variable load
operations modeling the expressions that denote the local variables and
parameters. The variable declarations are not part of the return statement, and
yet their modeled operations are present in the root expression graph.

> The operations modeling method parameter declarations occur twice, since
> the graph is rendered as tree.

To produce distinct root expressions graphs for each statement we need to do two
things, expand the set of root expression graphs, and prune the graphs by
removing the nodes corresponding to variable declaration operations that are not
directly associated with statements.

We expand the set of root expression graphs to include those whose operation is
a variable declaration operation, and more specifically only when the value used
to initialize the variable value is an operation result (thereby we avoid
including the variable values modeling the method parameters, where the value
used to initialize is a block parameter).

```java
List<Node<Value>> rootGraphs = graphs.values().stream()
        .filter(n -> n.value() instanceof Op.Result opr &&
                switch (opr.op()) {
                    // Variable declarations modeling local variables
                    case CoreOp.VarOp vop ->
                            vop.operands().get(0) instanceof Op.Result;
                    // An operation result with no uses
                    default -> opr.uses().isEmpty();
                })
        .toList();
```

We prune the graphs by enhancing method `expressionGraphs`, copying and
modifying, to create a new method `prunedExpressionGraphs`.

[//]: # (@formatter:off)
```java
static Map<Value, Node<Value>> prunedExpressionGraphs(CoreOp.FuncOp f) {
    return prunedExpressionGraphs(f.body());
}

static Map<Value, Node<Value>> prunedExpressionGraphs(Body b) {
    LinkedHashMap<Value, Node<Value>> graphs = new LinkedHashMap<>();
    // Traverse the model building structurally shared expression graphs
    b.elements().forEach(codeElement -> {
        switch (codeElement) {
            case Body _ -> { ... }
            case Block block -> { ... }
            // Prune graph for variable load operation
            case CoreOp.VarAccessOp.VarLoadOp op -> {
                // Ignore edge for the variable value operand
                graphs.put(op.result(), new Node<>(op.result(), List.of()));
            }
            // Prune graph for variable store operation
            case CoreOp.VarAccessOp.VarStoreOp op -> {
                // Ignore edge for the variable value operand
                // Add edge for value to store
                List<Node<Value>> edges = List.of(graphs.get(op.operands().get(1)));
                graphs.put(op.result(), new Node<>(op.result(), edges));
            }
            case Op op -> { ... }
        }
    });
    return graphs;
}

```
[//]: # (@formatter:on)

Two new cases are added checking if a code element is an instance of a variable
load or variable store operation respectively. For a variable load operation a
new node is created with no edges, since the single operand corresponds to the
variable value. For a variable store operation a new node is created with one
edge corresponding to the first operand, the value to store.

> Note that the switch is still exhaustive. The two new cases dominate the
> the more general operation case. Alternatively, we could have
> implemented the same behaviour within the more general operation case,
> checking if a dependent value is an instance of an operation result and the
> operation is an instance of a variable declaration. However, we think
> the above implementation is more instructive.

With these enhancements we can now compute three root expression graphs.

```text
@Reflect
static double squareDiff(double a, double b) {
    // a^2 - b^2 = (a + b) * (a - b)
    final double plus = a + b;
    final double minus = a - b;
    return plus * minus;
}

%7 : Var<java.type:"double"> = var %6 @loc="83:9" @"plus";
└── %6 : java.type:"double" = add %4 %5 @loc="83:29";
    ├── %4 : java.type:"double" = var.load %2 @loc="83:29";
    └── %5 : java.type:"double" = var.load %3 @loc="83:33";

%11 : Var<java.type:"double"> = var %10 @loc="84:9" @"minus";
└── %10 : java.type:"double" = sub %8 %9 @loc="84:30";
    ├── %8 : java.type:"double" = var.load %2 @loc="84:30";
    └── %9 : java.type:"double" = var.load %3 @loc="84:34";

%15 : java.type:"void" = return %14 @loc="85:9";
└── %14 : java.type:"double" = mul %12 %13 @loc="85:16";
    ├── %12 : java.type:"double" = var.load %7 @loc="85:16";
    └── %13 : java.type:"double" = var.load %11 @loc="85:23";
```

Notice how the three root expression graphs correspond, in order, to the three
statements in method `squareDiff`. These graphs are also _root expression
trees_, and it should also be possible to generate idiomatic C code from such
trees.

> The rules will need to be expanded if we want to support assignment
> expressions and distinguish them from assignment expression statements. We
> shall leave that investigation for another day.
