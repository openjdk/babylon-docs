# Automatic differentiation of Java code using Code Reflection
#### Paul Sandoz {.author}
#### February 2024 {.date}

In this article we will explain what is automatic differentiation, why it is
useful, and how we can use Code Reflection to help implement automatic
differentiation of Java methods.

Code Reflection is a Java platform feature being researched and developed under 
OpenJDK Project [Babylon](https://openjdk.org/projects/babylon/).

We will introduce Code Reflection concepts and APIs as we explain the problem
and present a solution. The explanations are neither exhaustive nor very
detailed, they are designed to give the reader an intuitive sense and
understanding of Code Reflection and its capabilities.

## Machine learning

Let's consider the field of machine learning, which is the science of producing
models that can predict what number is present in an image of a numerical digit,
or predicts what text a person is saying in an audio recording of human speech.

A machine learning model is really a mathematical function, `f` say (albeit a
possibly complex one with many inputs, outputs, and constants or parameters).

For a model to be effective it needs be trained, and many models are trained
using a *gradient descent algorithm*. This algorithm repeatedly applies known
inputs to `f`, comparing the computed output of `f` with expected output,
adjusting the parameters of `f`, until the difference between the computed
output and expected output, the error, is sufficiently reduced. The algorithm
adjusts the model by a gradient vector that is the result of applying the error
to the *differentiation* of `f`, `f'`

Machine learning developers implement such models using computer code, which
requires an implementation of `f` **and** its differential, `f'`, be available
for execution.

Consider the following method `f` implementing some mathematical function:

```
f(x, y) = x * (-sin(x * y) + y) * 4
```

The partial derivatives of `f` with respect to the input (or independent
variable) `x` can be manually calculated and written:

```jshelllanguage
df_dx(x, y) = (-sin(x * y) + y - x * cos(x * y) * y) * 4
```

(The partial derivative of `f` with respect to `y` is shown later as Java code.)

From the partial derivatives a gradient vector can be produced by combining the
results of calling partial derivative methods.

## Automatic differentiation

Manual differentiation is a very error-prone process. Although differentiation
is a mechanical process it's easy to make mistakes that are hard to debug, even
for a simple mathematical function as presented above, and the process quickly
becomes arduous as the mathematical function becomes more complex. This is
especially so for machine learning models. Computers are ideally suited to this
mechanical task.

To automatically differentiate a mathematical function written as a Java
method, `f` say, we need to write a Java program, `D` say, that implements the
rules of differentiation and applies those rules to a *symbolic representation*
of `f` to produce the differential method `f'`.

Program `D` can use *Code Reflection* to obtain a symbolic representation of
method `f`, called a *code model*. `D` can then traverse symbolic information
in `f`'s code model, *operations*, and apply the rules of differentiation to
those operations. For example, operations may be mathematical operations,
representing addition or multiplication, or invocation operations representing
invocations to Java methods that implement transcendental functions (such as
method `java.lang.Math::sin`).

`D` will produce a new code model, representing `f`', containing operations that
compute the differential, which can then be compiled to bytecode and invoked as
a Java program.

There are two approaches to automatic differentiation, forward-mode and
reverse-mode. For `N` independent variables, forward-mode automatic
differentiation requires the generation of `N` partial derivative methods, and
therefore `N` method calls to produce a gradient. Reverse-mode automatic
differentiation does not have these limitations, although it may be less
efficient for fewer independent variables.

Program `D` could be encapsulated in a Java library. We might ideally use it
like this:

```java

@CodeReflection
double f(double x, double y) {
    return x * (-sin(x * y) + y) * 4;
}

Function<double[], double[]> g_f = AD.gradientFunction(this::f);
double[] g = g_f.apply(new double[]{x, y});
```

We annotate our function to be differentiated, method `f`,
with `@CodeReflection`. This ensures there is a code model available for `f`,
and that it is accessible under similar access control rules as for its
invocation . Then we call the method `AD.gradientFunction` passing `f` as a
method reference. The method reference is targeted to a code reflection type
whose instance gives access to `f`'s code model.

How can the library author of method `gradientFunction` differentiate
method `f`?

## Implementing forward-mode automatic differentiation

In the following sections we will explain how to implement
the `gradientFunction` method using Code Reflection. We will focus on
forward-mode automatic differentiation since that is easier to understand, but
the same general principles could apply to reverse-mode automatic
differentiation.

A proof of concept implementation is available as a [test][ad-test] located in
the Babylon repository. The implementation is far from complete, and is just one
of many possible ways to approach the problem.

[ad-test]:https://github.com/openjdk/babylon/tree/code-reflection/test/jdk/java/lang/reflect/code/ad

## Differentiating simple functions

Let's focus on the simple mathematical function we presented earlier. It has two
independent variables, `x` and `y`.

```java

@CodeReflection
static double f(double x, double y) {
    return x * (-Math.sin(x * y) + y) * 4.0d;
}
```

It is annotated it with `@CodeReflection`. When it is compiled a code model will
be generated and made accessible at run time via reflection.

We can also write, by hand, the partial derivatives of `f` with respect to `x`
and `y` so we can test against what we generate.

```java
static double df_dx(double x, double y) {
    return (-Math.sin(x * y) + y - x * Math.cos(x * y) * y) * 4.0d;
}

static double df_dy(double x, double y) {
    return x * (1 - Math.cos(x * y) * x) * 4.0d;
}
```

### Obtaining a code model

Fundamentally, we need to implement a method that accepts a code model and a
reference to an independent variable to differentiate against, and produces a
new code model that is the partial derivative of the input. We will focus on
this aspect, although it is not as user-friendly as the prior example (
using program `D`, where we can observe the author only minimally used the Code
Reflection API to annotate their method).

First let's assume `f` is declared as a static method in a class `T`, say. To
obtain its code model using reflection we would do this.

```java
Method fm = T.class.getDeclaredMethod("f", double.class, double.class);
Optional<CoreOps.FuncOp> o = fm.getCodeModel();
CoreOps.FuncOp fcm = o.orElseThrow();
```

Using the reflection API we find the `java.lang.reflect.Method` instance of `f`,
and then ask it for its code model by invoking the method `getCodeModel`. Only
methods annotated with `@CodeReflection` will have code models, hence this
method is partial.

`f`'s code model is represented as an instance of `CoreOps.FuncOp`,
corresponding to a *function declaration* operation that models a Java method
declaration.

### Explaining the code model

A code model is a tree containing operations, bodies, and blocks. An operation
contains zero or more bodies. A body contains one or more blocks. A block
contains a sequence of one or more operations. A block can declare zero or more
block parameters, values. An operation declares an operation result, a value. An
operation may use values as operands, but only after they have been declared.

Using this simple tree structure we can define operations that model many Java
language constructs, and therefore we can build code models that model many Java
programs. This may appear surprising at first. Readers may be more familiar with
term "operation" in a more conventional sense, such as arithmetic operations.
However, given the structure described above, there is no need to limit
ourselves to this conventional sense. We are free to define an operation whose
operational semantics *declare* a function (instances of `CoreOps.FuncOp`),
model a Java lambda expression (instances of `CoreOps.LambdaOp`), or model a
Java `try` statement (instances of `ExtendedOps.JavaTryOp`).

What does the code model of `f` look like? We can serialize its in-memory form (
the instance of `CoreOps.FuncOp`) to a textual form.

```java
System.out.println(fcm.toText());
```

Which prints the following text.

```text
func @"f" (%0 : double, %1 : double)double -> {
    %2 : Var<double> = var %0 @"x";
    %3 : Var<double> = var %1 @"y";
    %4 : double = var.load %2;
    %5 : double = var.load %2;
    %6 : double = var.load %3;
    %7 : double = mul %5 %6;
    %8 : double = invoke %7 @"java.lang.Math::sin(double)double";
    %9 : double = neg %8;
    %10 : double = var.load %3;
    %11 : double = add %9 %10;
    %12 : double = mul %4 %11;
    %13 : double = constant @"4.0";
    %14 : double = mul %12 %13;
    return %14;
};
```

The textual form shows the code model's root is a function declaration (`func`)
operation. The function declaration operation has an operation result, like all
other operations, but since it's the root of the tree there is no need to
present it.

The lambda-like expression represents the fusion of the function declaration
operation's single body and the body's first and only block, called the entry
block. Then there is a sequence of operations in the entry block. For each
operation there is an instance of a corresponding class present in the in-memory
form, all of which extend from the abstract class `java.lang.reflect.code.Op`.

The entry block has two block parameters, `%0` and `%1` (corresponding to `x`
and `y`), each described by a type of `double`, which model `f`'s method
parameters. These parameters are used as operands of various operations. Many
operations produce operation results, e.g., `%12` the result of a multiplication
operation, that are used as operands of subsequent operations, and so on.
The `return` operation has a result, again like all other operations, but since
that result cannot be meaningfully used we don't present it.

Code models have the property of Static Single-Assignment (SSA). We refer to
variables that can only be assigned once as values (they are a bit like final
variables in Java) .e.g., value `%12` can never be modified. A variable
declaration is modeled as an operation that produces a value that holds a
value (a box), and access operations load or store to that box.

(Some readers may be thinking this looks very similar
to [MLIR](https://mlir.llvm.org/) and that is by design.)

We can see how the operations model Java language constructs like method
declarations, variables (method parameters or local variables) and access of
variables, binary and unary mathematical operations, or method invocations (
e.g., to method `java.lang.Math::sin`).

### Analyzing the model

We can simplify this model by transforming it into one that removes the variable
declarations and accesses. We call this a pure SSA transform.

```jshelllanguage
fcm = SSA.transform(fcm);
```

The textual form of the resulting code model is as follows.

```text
func @"f" (%0 : double, %1 : double)double -> {
    %2 : double = mul %0 %1;
    %3 : double = invoke %2 @"java.lang.Math::sin(double)double";
    %4 : double = neg %3;
    %5 : double = add %4 %1;
    %6 : double = mul %0 %5;
    %7 : double = constant @"4.0";
    %8 : double = mul %6 %7;
    return %8;
};
```

This is a simpler model, where program meaning is preserved. Since the model is
simpler it becomes simpler to analyze in preparation for automatic
differentiation.

When differentiating with respect to an independent variable we will analyze the
model with respect to that variable and compute an *active set* of values, those
which depend transitively on the independent variable.

For example the block parameter `%0` (representing the independent variable `x`)
is used as an operand by the operation that produces the operation result `%2` (
the result of a multiplication), and so on.

The active set for `%0` is `{%0, %2, %3, %4, %5, %6, %8, %9}`. Note the
value `%9` represents the result of the `return` operation, whose type
is `void`, and whose value is not explicitly shown in the textual form.

The active set for `%1` (representing the independent variable `y`) is the same
in this case.

We can compute this set by traversing the *uses* of values in the code model.
The in-memory representation of a code model is constructed such that the uses
are easily available.

A simple and naive implementation might be as follows.

```java
static Set<Value> activeSet(Value root) {
    Set<Value> s = new LinkedHashSet<>();
    activeSet(s, root);
    return s;
}

static void activeSet(Set<Value> s, Value v) {
    s.add(v);
    // Iterate over the uses of v
    for (Op.Result use : v.uses()) {
        activeSet(s, use);
    }
}
```

In practice the active set computation needs to be a little more complex, as
implemented in the test code and used later in an example that has mathematical
expressions embedded within some control flow.

### Reporting programming errors

In general not all Java programs are differentiable, it's a constrained
programming model. The author of a Java method, to be differentiated, needs to
be aware of the constraints, and the automatic differentiation program should
report programming errors.

After computing the active set (or during that computation) we could perform
checks and report errors if values cannot be differentiated or if the
independent value does not contribute to a result etc. Further, we might want to
do additional checks on the model and fail if unsupported language constructs
are present e.g., perhaps try statements, or ignore other constructs. The test
code does not perform such extensive checks.

Ideally the reporting of such errors would occur when the method is compiled by
the source compiler rather than at runtime. Code reflection can also make
available the same code model at compile time for such purposes, but we will not
explore this capability in this article.

### Differentiating a code model

Once we have the active set we can use it to drive differentiation of the code
model. The operation results in the active set refer to the operations we need
to differentiate.

We can *transform* the code model, applying the rules of differentiation to the
required operations we encounter.

Code models are immutable. Code models can be produced by building, or by
transforming an existing code model. Transforming takes an input code model and
builds an output code model. For each input operation encountered in the input
code model we have the choice to add that operation to the builder of the output
code model (copying), to not add it (removal), or add new output operations
(replacement or addition).

First lets declare a class, `ForwardDifferentiation`, whose constructor computes
the active set.

```java
import static java.lang.reflect.code.op.CoreOps.*;

public final class ForwardDifferentiation {
    // The function to differentiate
    final FuncOp fcm;
    // The independent variable
    final Block.Parameter ind;
    // The active set for the independent variable
    final Set<Value> activeSet;
    // The map of input value to it's (output) differentiated value
    final Map<Value, Value> diffValueMapping;

    // The constant value 0.0d
    // Declared in the (output) function's entry block
    Value zero;

    private ForwardDifferentiation(FuncOp fcm, Block.Parameter ind) {
        int indI = fcm.body().entryBlock().parameters().indexOf(ind);
        if (indI == -1) {
            throw new IllegalArgumentException("Independent argument not defined by function");
        }
        this.fcm = fcm;
        this.ind = ind;

        // Calculate the active set of dependent values for the independent value
        this.activeSet = ActiveSet.activeSet(fcm, ind);
        // A mapping of input values to their (output) differentiated values
        this.diffValueMapping = new HashMap<>();
    }
}
```

A map from input operation results to (output) differentiated
values, `diffValueMapping` is also constructed. This will be used to keep track
of differentiated values that may be used in dependent computations. Because the
code model is immutable there is no danger that values referred to in the input
code model will become stale or be modified.

Next we declare a static factory method on `ForwardDifferentiation` to compute
the partial derivative:

```java
public static FuncOp partialDiff(FuncOp fcm, Block.Parameter ind) {
    return new ForwardDifferentiation(fcm, ind).partialDiff();
}
```

This static method constructs an instance of `ForwardDifferentiation` and calls
the method `partialDiff`, which performs the transformation.

```java
FuncOp partialDiff() {
    int indI = fcm.body().entryBlock().parameters().indexOf(ind);

    // Transform f to f' w.r.t ind
    AtomicBoolean first = new AtomicBoolean(true);
    FuncOp dfcm = fcm.transform(STR."d\{fcm.funcName()}_darg\{indI}",
            (block, op) -> {
                // Initialization
                if (first.getAndSet(false)) {
                    // Declare the zero value constant
                    zero = block.op(constant(ind.type(), 0.0d));
                    // Declare the one value constant
                    Value one = block.op(constant(ind.type(), 1.0d));
                    // The differential of ind is one
                    // For all other parameters it is zero (absence from the map)
                    diffValueMapping.put(ind, one);
                }

                // If the result of the operation is in the active set,
                // then differentiate it, otherwise copy it
                if (activeSet.contains(op.result())) {
                    Value dor = diffOp(block, op);
                    // Map the input result to its (output) differentiated result
                    // so that it can be used when differentiating subsequent operations
                    diffValueMapping.put(op.result(), dor);
                } else {
                    // Block is not part of the active set, just copy it
                    block.op(op);
                }
                return block;
            });

    return dfcm;
}
```

We transform the code model using the `FuncOp` operation's `transform` method.
It accepts a name for the function of the output code model, and a lambda
expression that accepts an (output) block builder and an input operation. The
transform method will traverse all operations in the input code model and report
encountered input operations to the lambda expression.

On first encounter we declare a few constant values in the output model by
adding constant operations, instances of `ConstantOp`, to the output model.
Specifically, we declare the zero constant value of `0.0d`, which will be used
as an operand of subsequent operations we add to output model.

On each encounter, if the operation's result is a member of the active set then
we differentiate it, and map the (input) result to its (output) differential
value. We apply the chain rule from inside to outside.

The method `diffOp` applies the rules of differentiation. The method consists of
a switch expression with pattern matching whose target is the instance of the
operation to differentiate. We show below a subset of interesting switch cases
from that method.

```java
Value diffOp(Block.Builder block, Op op) {
    // Switch on the op, using pattern matching
    return switch (op) {
        case ... -> {
        }
        case CoreOps.MulOp _ -> {
            // Copy input operation
            block.op(op);

            // Product rule
            // diff(l) * r + l * diff(r)
            Value lhs = op.operands().get(0);
            Value rhs = op.operands().get(1);
            Value dlhs = diffValueMapping.getOrDefault(lhs, zero);
            Value drhs = diffValueMapping.getOrDefault(rhs, zero);
            Value outputLhs = block.context().getValue(lhs);
            Value outputRhs = block.context().getValue(rhs);
            yield block.op(add(
                    block.op(mul(dlhs, outputRhs)),
                    block.op(mul(outputLhs, drhs))));
        }
        case CoreOps.InvokeOp c -> {
            MethodDesc md = c.invokeDescriptor();
            String operationName = null;
            if (md.refType().equals(J_L_MATH)) {
                operationName = md.name();
            }
            // Differentiate sin(x)
            if ("sin".equals(operationName)) {
                // Copy input operation
                block.op(op);

                // Chain rule
                // cos(expr) * diff(expr)
                Value a = op.operands().get(0);
                Value da = diffValueMapping.getOrDefault(a, zero);
                Value outputA = block.context().getValue(a);
                Op.Result cosx = block.op(invoke(J_L_MATH_COS, outputA));
                yield block.op(mul(cosx, da));
            } else {
                throw new UnsupportedOperationException("Operation not supported: " + op.opName());
            }
        }
    };
}
```

The first case shows how we compute the differential of a multiply operation, an
instance of `MulOp`, by applying the product rule.

We add the input multiply operation to the builder, which copies it from the
input model to the output model, since its result might be used in further
computations added to the output model.

Given the input values for the first (left) and second (right) operands of the
multiply operation we obtain their differentiated values, which must have been
computed earlier, or if not then they must be zero. Then we add two new multiply
operations corresponding to the product rule. To do that we also need to obtain
the output values that map to the input values of the first and second
operands (again these must have been computed earlier when copying prior input
operations). Finally, we sum the results of the multiplications with an add
operation and yield the result.

The second cases shows how we compute the differential of `sin(x)` (which
is `cos(x)x'`). We match on an invoke operation, an instance of `InvokeOp`, that
invokes the method `java.lang.Math::sin`. We copy the invocation operation, add
an invocation operation to invoke `java.lang.Math::cos`, then add a multiply
operation, the result of which is yielded.

We anticipate patterns and switches will be used for many kinds transformation
and are designing the code reflection APIs with this in mind. We are looking
forward to future language capabilities where we can write our own patterns,
which will enable more sophisticated tree-based matching of operations (
including on their uses or their operands).

Let's piece it all together and print out the textual form of the differentiated
code model.

```java
import ad.ForwardDifferentiation;

Method fm = T.class.getDeclaredMethod("f", double.class, double.class);
Optional<CoreOps.FuncOp> o = fm.getCodeModel();
CoreOps.FuncOp fcm = SSA.transform(o.orElseThrow());
Block.Parameter x = fcm.body().entryBlock().parameters().get(0);
// Code model in, code model out
CoreOps.FuncOp dfcm_x = ForwardDifferentiation.partialDiff(fcm, x);
```

```text
func @"df_darg0" (%0 : double, %1 : double)double -> {
    %2 : double = constant @"0.0";
    %3 : double = constant @"1.0";
    %4 : double = mul %0 %1;
    %5 : double = mul %3 %1;
    %6 : double = mul %0 %2;
    %7 : double = add %5 %6;
    %8 : double = invoke %4 @"java.lang.Math::sin(double)double";
    %9 : double = invoke %4 @"java.lang.Math::cos(double)double";
    %10 : double = mul %9 %7;
    %11 : double = neg %8;
    %12 : double = neg %10;
    %13 : double = add %11 %1;
    %14 : double = add %12 %2;
    %15 : double = mul %0 %13;
    %16 : double = mul %3 %13;
    %17 : double = mul %0 %14;
    %18 : double = add %16 %17;
    %19 : double = constant @"4.0";
    %20 : double = mul %15 %19;
    %21 : double = mul %18 %19;
    %22 : double = mul %15 %2;
    %23 : double = add %21 %22;
    return %23;
};
```

and compare against the handwritten Java code:

```java
static double df_dx(double x, double y) {
    return (-Math.sin(x * y) + y - x * Math.cos(x * y) * y) * 4.0d;
}
```

We can immediately see the differentiated code model has more mathematical
operations, many are redundant e.g., there are multiplications by 0 or 1, and
the result of one operation (`%20`) is not used.

We need to transform the code model further to remove redundant operations (
commonly referred to as expression elimination). The resulting code model with
eliminated expressions is show below.

(We will not get into the specifics of how expression elimination is
implemented. It uses the same code reflection APIs and similar techniques we
have shown above, and the curious reader may be interested in looking at the
code.)

```text
func @"df_darg0" (%0 : double, %1 : double)double -> {
    %2 : double = constant @"0.0";
    %3 : double = mul %0 %1;
    %4 : double = add %1 %2;
    %5 : double = invoke %3 @"java.lang.Math::sin(double)double";
    %6 : double = invoke %3 @"java.lang.Math::cos(double)double";
    %7 : double = mul %6 %4;
    %8 : double = sub %1 %5;
    %9 : double = sub %2 %7;
    %10 : double = mul %0 %9;
    %11 : double = add %8 %10;
    %12 : double = constant @"4.0";
    %13 : double = mul %11 %12;
    %14 : double = add %13 %2;
    return %14;
};
```

We could eliminate some expressions in the differentiation transformation, but
this will complicate the transformation. Often it is better to keep
transformations focused, and separate into two or more transformation stages, at
the potential expense of more work.

From this code model we can interpret it or translate to bytecode, and execute
it by invoking a method handle.

If we apply the same to the independent variable `y` we will get two code
models, from which we can compute the gradient, and therefore implement the
`gradientFunction` method.

## Differentiating models with control flow

In the prior example we differentiated a Java method implementing a simple
mathematical expression. In this section we will consider a more complex method
that contains mathematical expressions embedded in control flow statements.

```java

@CodeReflection
static double f(/* independent */ double x, int y) {
    /* dependent */
    double o = 1.0;
    for (int i = 0; i < y; i = i + 1) {
        if (i > 1) {
            if (i < 5) {
                o = o * x;
            }
        }
    }
    return o;
}
```

Method `f` has one independent variable, `x`. The parameter `y` indirectly
affects computations using `x`, but there is no direct data dependency between
`x` and `y`. We can see a multiplication operation embedded within a loop and
some conditional control flow. This method can be differentiating using the same
techniques we have presented.

Here is the hand-coded version.

```java
static double df_dx(/* independent */ double x, int y) {
    double d_o = 0.0;
    double o = 1.0;
    for (int i = 0; i < y; i = i + 1) {
        if (i > 1) {
            if (i < 5) {
                d_o = d_o * x + o * 1.0;
                o = o * x;
            }
        }
    }
    return d_o;
}
```

Notice the product rule applied to the multiplication of `o * x`, and in source
we have to be careful that this computation is applied before `o` is updated.

(Astute readers may observe that and `o` and `d_o`, and `x` and `1`, correspond
to the two components of
a [dual number](https://en.wikipedia.org/wiki/Dual_number#Differentiation).)

`f`'s code model will contain operations modeling the `for` loop and `if`
statements, retaining the structure of the code.

```text
func @"f" (%0 : double, %1 : int)double -> {
    %2 : Var<double> = var %0 @"x";
    %3 : Var<int> = var %1 @"y";
    %4 : double = constant @"1.0";
    %5 : Var<double> = var %4 @"o";
    java.for
        ()Var<int> -> {
            %6 : int = constant @"0";
            %7 : Var<int> = var %6 @"i";
            yield %7;
        }
        (%8 : Var<int>)boolean -> {
            %9 : int = var.load %8;
            %10 : int = var.load %3;
            %11 : boolean = lt %9 %10;
            yield %11;
        }
        (%12 : Var<int>)void -> {
            %13 : int = var.load %12;
            %14 : int = constant @"1";
            %15 : int = add %13 %14;
            var.store %12 %15;
            yield;
        }
        (%16 : Var<int>)void -> {
            java.if
                ()boolean -> {
                    %17 : int = var.load %16;
                    %18 : int = constant @"1";
                    %19 : boolean = gt %17 %18;
                    yield %19;
                }
                ()void -> {
                    java.if
                        ()boolean -> {
                            %20 : int = var.load %16;
                            %21 : int = constant @"5";
                            %22 : boolean = lt %20 %21;
                            yield %22;
                        }
                        ()void -> {
                            %23 : double = var.load %5;
                            %24 : double = var.load %2;
                            %25 : double = mul %23 %24;
                            var.store %5 %25;
                            yield;
                        }
                        ()void -> {
                            yield;
                        };
                    yield;
                }
                ()void -> {
                    yield;
                };
            java.continue;
        };
    %26 : double = var.load %5;
    return %26;
};
```

We can clearly see that some operations contain bodies at many levels. In this
case each body has one (entry) block as in the prior example. The four bodies in
the `for` operation correspond to the nested expressions and statements as
specified by the Java Language Specification.

Rather than modifying the `diffOp` method to know about such operations and
their behaviour we can instead simplify and generalize the solution. Code
Reflection defines two sets of operations. Core operations which can be used to
model a wide set of Java programs, and extended (or auxiliary) operations. The
extended operations model Java langauge constructs such as `for`
statements, `if` statements, and `try` statements. The extended operations may
be lowered into a sequence of core operations within a connected set of blocks
within a body.

We can lower `f`'s code model using a lowering transformation and then transform
to pure SSA.

```text
fcm = fcm.transform((block, op) -> {
    if (op instanceof Op.Lowerable lop) {
        return lop.lower(block);
    } else {
        block.op(op);
        return block;
    }
});
fcm = SSA.transform(fcm);
```

```text
func @"fcf" (%0 : double, %1 : int)double -> {
    %2 : double = constant @"1.0";
    %3 : int = constant @"0";
    branch ^block_0(%2, %3);
  
  ^block_0(%4 : double, %5 : int):
    %6 : boolean = lt %5 %1;
    cbranch %6 ^block_1 ^block_2;
  
  ^block_1:
    %7 : int = constant @"1";
    %8 : boolean = gt %5 %7;
    cbranch %8 ^block_3 ^block_4;
  
  ^block_3:
    %9 : int = constant @"5";
    %10 : boolean = lt %5 %9;
    cbranch %10 ^block_5 ^block_6;
  
  ^block_5:
    %11 : double = mul %4 %0;
    branch ^block_7(%11);
  
  ^block_6:
    branch ^block_7(%4);
  
  ^block_7(%12 : double):
    branch ^block_8(%12);
  
  ^block_4:
    branch ^block_8(%4);
  
  ^block_8(%13 : double):
    branch ^block_9;
  
  ^block_9:
    %14 : int = constant @"1";
    %15 : int = add %5 %14;
    branch ^block_0(%13, %15);
  
  ^block_2:
    return %4;
};
```

We can see that the resulting code model (program meaning is preserved) has
multiple blocks within the `func` operation's body. There is no longer any
nesting of bodies.

Control flow of the program is modelled by the blocks and how they are connected
together to form a *control flow graph*. In `block_9` we can observe a branch
back to `block_0` which passes values of `o` and `i` as block arguments for the
next loop iteration. (Block arguments and parameters are analogous to phi nodes
in other approaches that model code.)

We need to extend our implementation, computing the active set and
differentiating operations, to understand these connections between blocks.

With some modest enhancements we can compute the active set for `x` and
differentiate this Java method. The active set will have block parameters as
members, such as parameter `%4`, which represents the value `o` for the current
loop iteration (since `o` is dependent on `x`). The differentiated model is
presented below.

```text
func @"dfcf_darg0" (%0 : double, %1 : int)double -> {
    %2 : double = constant @"0.0";
    %3 : double = constant @"1.0";
    %4 : double = constant @"1.0";
    %5 : int = constant @"0";
    branch ^block_0(%4, %5, %2);
  
  ^block_0(%6 : double, %7 : int, %8 : double):
    %9 : boolean = lt %7 %1;
    cbranch %9 ^block_1 ^block_2;
  
  ^block_1:
    %10 : int = constant @"1";
    %11 : boolean = gt %7 %10;
    cbranch %11 ^block_3 ^block_4;
  
  ^block_3:
    %12 : int = constant @"5";
    %13 : boolean = lt %7 %12;
    cbranch %13 ^block_5 ^block_6;
  
  ^block_5:
    %14 : double = mul %6 %0;
    %15 : double = mul %8 %0;
    %16 : double = mul %6 %3;
    %17 : double = add %15 %16;
    branch ^block_7(%14, %17);
  
  ^block_6:
    branch ^block_7(%6, %8);
  
  ^block_7(%18 : double, %19 : double):
    branch ^block_8(%18, %19);
  
  ^block_4:
    branch ^block_8(%6, %8);
  
  ^block_8(%20 : double, %21 : double):
    branch ^block_9;
  
  ^block_9:
    %22 : int = constant @"1";
    %23 : int = add %7 %22;
    branch ^block_0(%20, %23, %21);
  
  ^block_2:
    return %8;
};
```

We can observe in `block_5` the application of the product rule. Further we can
observe additional block arguments and parameters required to pass along the
differential of `o`, `d_o`. In `block_9` there is a branch with an additional
block argument appended, value `%21`, that becomes the next value of `d_o`
in `block_0`, parameter `%8`
