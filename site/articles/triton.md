# Exploring Triton GPU programming for neural networks in Java

#### Paul Sandoz {.author}

#### February 2024 {.date}, Updated Dec 2025 {.date}

In this article we will explain how we can use Code Reflection to implement the
[Triton][Triton-intro] programming model in Java as an alternative to Python.

Code Reflection is a Java platform feature being researched and developed under
OpenJDK Project [Babylon](https://openjdk.org/projects/babylon/).

We will introduce Code Reflection concepts and APIs as we explain the problem
and present a solution. The explanations are neither exhaustive nor very
detailed, they are designed to give the reader an intuitive sense and
understanding of Code Reflection and its capabilities.

## Triton

[Triton][Triton-intro] is a domain-specific programming model that compiler
developers can use to write programs in Python that compile to GPU code.

Triton enables developers with little or no experience of GPU hardware and
GPU-specific programming languages, such as CUDA, to write very efficient
parallel programs.

[Triton-intro]: https://triton-lang.org/main/programming-guide/chapter-1/introduction.html

The release announcement for Triton [states][Triton-blog]:

> Triton makes it possible to reach peak hardware performance with relatively
> little effort; for example, it can be used to write FP16 matrix multiplication
> kernels that match the performance of cuBLAS—something that many GPU
> programmers can’t do—in under 25 lines of code. Our researchers have already
> used it to produce kernels that are up to 2x more efficient than equivalent
> Torch implementations, and we’re excited to work with the community to make
> GPU programming more accessible to everyone.

[Triton-blog]: https://openai.com/research/triton

The Triton programming model hides the thread-based programming model of CUDA.
Thereby, the Triton compiler is able to better leverage the GPU hardware e.g.,
such as optimizing for cases that might otherwise require explicit
synchronization.

To enable this abstraction the developer programs against a Triton Python API,
where arithmetic computations are performed on tensors rather than scalars. Such
tensors are required to have constant shape, the number of dimensions and their
size must be constant (in addition the size must be a power of two).

### Vector addition

To explain the programming model we shall present a simple example, vector
addition. This example is instructive even though it can be easily written in
CUDA.

The complete example is presented as a [tutorial][tutorial-vector-addition] at
the Triton website, including how Triton integrates with PyTorch. We shall focus
on the Triton program.

[tutorial-vector-addition]: https://triton-lang.org/main/getting-started/tutorials/01-vector-add.html#sphx-glr-getting-started-tutorials-01-vector-add-py

```python
import triton
import triton.language as tl


@triton.jit
def add_kernel(x_ptr,  # *Pointer* to first input vector.
               y_ptr,  # *Pointer* to second input vector.
               output_ptr,  # *Pointer* to output vector.
               n_elements,  # Size of the vector.
               BLOCK_SIZE: tl.constexpr,
               # Number of elements each program should process.
               # NOTE: `constexpr` so it can be used as a shape value.
               ):
    # There are multiple 'programs' processing different data. We identify which program
    # we are here:
    pid = tl.program_id(axis=0)  # We use a 1D launch grid so axis is 0.
    # This program will process inputs that are offset from the initial data.
    # For instance, if you had a vector of length 256 and block_size of 64, the programs
    # would each access the elements [0:64, 64:128, 128:192, 192:256].
    # Note that offsets is a list of pointers:
    block_start = pid * BLOCK_SIZE
    offsets = block_start + tl.arange(0, BLOCK_SIZE)
    # Create a mask to guard memory operations against out-of-bounds accesses.
    mask = offsets < n_elements
    # Load x and y from DRAM, masking out any extra elements in case the input is not a
    # multiple of the block size.
    x = tl.load(x_ptr + offsets, mask=mask)
    y = tl.load(y_ptr + offsets, mask=mask)
    output = x + y
    # Write x + y back to DRAM.
    tl.store(output_ptr + offsets, output, mask=mask)
```

The comments are quite informative, and we recommend taking a few moments to
read them carefully. The program is annotated with `@triton.jit`, which
identifies it as a Triton program.

This program is designed to be compiled to a GPU program and executed multiple
times in parallel on the GPU. Each execution will have a program identifier
associated with it, obtained by calling the Triton language API
method `program_id`. This is not a thread identifier, although developers
familiar with CUDA will likely recognize that it is used in similar manner.

The program identifier is used to locate the start index in the input and output
vectors from which to compute on. The end index is determined by the method
parameter, `BLOCK_SIZE`. Notice that this is annotated with `tl.constexpr`. When
the program is compiled it is required that `BLOCK_SIZE` be passed as a constant
value that is a power of two. Therefore, the interval of computation
is `[pid * BLOCK_SIZE, pid * BLOCK_SIZE + BLOCK_SIZE)`. Program execution will
be arranged such that program identifiers are proportioned according to the
total size of the computation with respect to `BLOCK_SIZE`.

The program computes the start index, held in the scalar variable `block_start`,
but does not compute the end index. Instead, the program creates a _tensor_, by
calling the method `tl.arange`:

```python
tl.arange(0, BLOCK_SIZE)
```

This method creates a tensor of one dimension whose size is `BLOCK_SIZE`. The
elements of the tensor are 32-bit integers, and they are initialized to range
from `0` to `BLOCK_SIZE - 1` consecutively. If, for example, the program was
compiled with `BLOCK_SIZE=64` then we know the tensor's shape i.e., we know the
_type_ of the tensor. This is a very important property. It means we can
statically check that expressions with tensors are type safe with respect to
their shape, and if necessary insert operations to convert tensors and scalars
(known as splatting or broadcasting). (In this case we also know the tensor's
elements are constant.)

The result of that tensor is then input to the following arithmetic expression
which computes offsets into the vectors (pointed to by `x_ptr` etc.).

```python
offsets = block_start + tl.arange(0, BLOCK_SIZE)
```

Python's dynamic typing and flexible operator overloading allows the program to
express the addition of a scalar with a tensor. Since we know the type of the
tensor we can convert the scalar `block_start` to a tensor of the same type and
whose elements all have the same value as `block_start`. This is generally
referred to as a _broadcasting_ operation (or sometimes referred to as splatting
when broadcasting scalars). Then we can add the two tensors together, resulting
in the `offsets` tensor. We know this is a 1D tensor of size `BLOCK_SIZE`, whose
elements range from `[block_start, block_start + BLOCK_SIZE)`.

The `offsets` tensor may reference vector elements that are out of bounds i.e.
values greater than the size of the input and output vectors, `n_elements`. To
protect against out of bounds access the program creates a tensor _mask_.

```python
mask = offsets < n_elements
```

Again, Python's dynamic typing allows the program to compare a tensor with a
scalar value. Similarly to the prior addition, we can broadcast the value of
`n_elements` to a tensor with the same type as `offsets`. The elements of each
tensor at the same index are compared, producing a `0` or `1` element at the
same index in the resulting `mask` tensor if the comparison returned `false`
or `true` respectively.

Given the `offsets` and `mask` tensors the program can safely load tensors from
memory pointed to by the pointers `x_ptr` and `y_ptr`.

```python
x = tl.load(x_ptr + offsets, mask=mask)
y = tl.load(y_ptr + offsets, mask=mask)
```

If `x_ptr` points to 32-bit floating point values, the
expression `x_ptr + offsets` results in a tensor of pointers to 32-bit floating
point values (and similarly for `y_ptr`). We know the pointers are contiguous in
memory, in the
interval `[x_ptr + block_start, x_ptr + block_start + BLOCK_SIZE)`.

The `tl.load` method will load a tensor of values from memory pointed to by
tensor of pointers. The resulting tensor has the same shape as the tensor of
pointers, and a zero value will be placed in the resulting tensor for any
corresponding zero mask value (for access that is out of bounds).

The program can then add the two floating point tensors together and store the
result.

### Triton compiler

The Triton compiler is responsible for compiling a Triton program (a program
written in Python and annotated with `@triton.jit`) to a GPU program, commonly
referred to as a kernel.

The stages of the compiler can be broadly described in the following diagram.

```text
    Python program
      |
      |  AST visitor
      V
    Triton MLIR
      |
      |  Triton MLIR compiler
      V
     PTX
```

The Triton compiler transforms the Python program to Triton MLIR which is then
compiled to PTX by the native Triton MLIR compiler.

The Multi-Level IR Compiler Framework ([MLIR][MLIR]) provides reusable and
extensible compiler infrastructure. It defines a metamodel for representing and
transforming code, with corresponding C/C++ APIs and modular infrastructure for
building compilers. Program functionality is specified by MLIR _dialects_
that define a set of types and operations e.g., such as types and operations
associated with linear algebra.

[MLIR]:https://mlir.llvm.org/

The Triton compiler supports a set of MLIR [dialects][Triton-dialects], which
define types and operations specific to Triton programs. The Triton dialect uses
and depends on other MLIR dialects. For example, it uses the
[arith][arith-dialect] dialect and the [ranked tensor type][tensor-type] of the
builtin dialect. Reusing dialects means the Triton MLIR compiler can reuse
existing compiler infrastructure to compile Triton MLIR. In fact, the Triton
MLIR compiler is itself likely composed of multiple transformations that
progressively lower the program to PTX code.

[Triton-dialects]:https://triton-lang.org/main/dialects/dialects.html

[arith-dialect]:https://mlir.llvm.org/docs/Dialects/ArithOps/

[tensor-type]:https://mlir.llvm.org/docs/Dialects/Builtin/#rankedtensortype

Transformation to Triton MLIR is performed by visiting the AST of the Python
program. The AST visitor is responsible for type checking the Triton program.
This will include ensuring all tensors are of known shape (derived from
constants input to the compiler, as explained in the next section), and
expressions using tensors are shape compatible. (Some aspects of this were
mentioned when describing the vector addition example.)

Given type correct tensor expressions the AST visitor can build a Triton MLIR
program, inserting appropriate tensor operations and broadcasting conversions.

### MLIR of a Triton program

We can execute the Triton compiler against the prior Triton program and view the
Triton MLIR program. (Note that we had to make a few modifications to the Triton
compiler to print out the Triton MLIR program in the absence of available CUDA
software and supporting hardware.)

```shell
python3 python/triton/tools/compile.py \
  --kernel-name add_kernel \
  --signature "*fp32,*fp32,*fp32,i32,64" \
  --grid=1024,1024,1024 \
  python/tutorials/01-vector-add.py
```

The compiler requires a signature of the kernel that declares the types of the
Triton program's parameters. Thereby, the compiler can perform static type
checking. Notice the string "64" represents a constant type of integer whose
value is 64, and is associated with the constant parameter `BLOCK_SIZE`.

The textual-form of the Triton MLIR program is as follows:

```mlir
module {
  tt.func public @add_kernel_0123(%arg0: !tt.ptr<f32, 1> ,
                                  %arg1: !tt.ptr<f32, 1> ,
                                  %arg2: !tt.ptr<f32, 1> ,
                                  %arg3: i32 )
                                  attributes {noinline = false} {
    %0 = tt.get_program_id x : i32
    %c64_i32 = arith.constant 64 : i32
    %1 = arith.muli %0, %c64_i32 : i32
    %2 = tt.make_range {end = 64 : i32, start = 0 : i32} : tensor<64xi32>
    %3 = tt.splat %1 : (i32) -> tensor<64xi32>
    %4 = arith.addi %3, %2 : tensor<64xi32>
    %5 = tt.splat %arg3 : (i32) -> tensor<64xi32>
    %6 = arith.cmpi slt, %4, %5 : tensor<64xi32>
    %7 = tt.splat %arg0 : (!tt.ptr<f32, 1>) -> tensor<64x!tt.ptr<f32, 1>>
    %8 = tt.addptr %7, %4 : tensor<64x!tt.ptr<f32, 1>>, tensor<64xi32>
    %9 = tt.load %8, %6 {cache = 1 : i32, evict = 1 : i32, isVolatile = false} : tensor<64xf32>
    %10 = tt.splat %arg1 : (!tt.ptr<f32, 1>) -> tensor<64x!tt.ptr<f32, 1>>
    %11 = tt.addptr %10, %4 : tensor<64x!tt.ptr<f32, 1>>, tensor<64xi32>
    %12 = tt.load %11, %6 {cache = 1 : i32, evict = 1 : i32, isVolatile = false} : tensor<64xf32>
    %13 = arith.addf %9, %12 : tensor<64xf32>
    %14 = tt.splat %arg2 : (!tt.ptr<f32, 1>) -> tensor<64x!tt.ptr<f32, 1>>
    %15 = tt.addptr %14, %4 : tensor<64x!tt.ptr<f32, 1>>, tensor<64xi32>
    tt.store %15, %13, %6 {cache = 1 : i32, evict = 1 : i32} : tensor<64xf32>
    tt.return
  }
}
```

MLIR programs have the property of Static Single-Assignment (SSA). We refer to
variables that can only be assigned once as values (they are a bit like final
variables in Java) e.g., value `%0` can never be modified.

Notice that there are only four parameters corresponding to values `%arg0`
to `%arg3`: three pointers to 32-bit floats corresponding to the two input
vectors and the output vector; and the 32-bit integer corresponding to the size
of the vectors. The constant parameter has been folded away.

The Python method call to `tl.arange` has been transformed to the Triton MLIR
operation `tt.make_range`. The constant values passed to the Python method (the
constant 0, and constant `BLOCK_SIZE` whose value is 64) are present as the
operation's attributes. The operation returns a value, `%2`, whose type
is `tensor<64xi32>`, a ranked tensor of one dimension with a size of 64, and
whose elements are 32-bit integers.

The addition in the Python expression `block_start + tl.arange(0, BLOCK_SIZE)`
has been transformed into two operations:

```
%3 = tt.splat %1 : (i32) -> tensor<64xi32>
%4 = arith.addi %3, %2 : tensor<64xi32>
```

Before performing the addition the scalar value is converted to a tensor
("splatting"). If we carefully follow the use of values in subsequent operations
we can observe that all the tensors are of constant shape.

## Triton programs in Java

We will show that it is feasible for developers to write Triton programs in Java
that are surprisingly comparable to the Triton programs in Python, and have the
potential to compile to PTX code. The focus will be on the front-end of a Java
Triton compiler that has the following stages.

```text
    Java program
      |
      |  Code reflection
      V
    Java code model
      |
      |  Code model transformer
      V
    Triton code model
```

Using Code Reflection we can obtain a _symbolic representation_ of the Java
program, called a Java *code model*. We can then traverse the symbolic
information in the Java code model, *operations* modeling Java program
behaviour, and apply the rules of the Triton programming model to produce a
Triton code model that contains operations modeling Triton program behaviour.
Note that this transformation will not preserve Java program meaning. The
resulting Triton code model is not a Java program and will not be executed by
the Java runtime.

The Triton code model can then be input to subsequent compiler stages that
transform it to PTX. We shall not focus on that aspect. In theory, it should be
possible to reuse the existing native Triton MLIR compiler, first transforming
the Triton code model to Triton MLIR.

We shall see later that code models have similar properties to MLIR. That is by
design. One goal of Code Reflection is to enable interoperation with and reuse
of native compiler tool chains. In combination with the Foreign Function and
Memory API we have the potential to interact natively with MLIR APIs and
toolchains. Triton programming in Java represents an important use-case in this
respect.

A proof of concept Java Triton API and front-end compiler has been implemented,
and is located in the Babylon repo [here][java-triton].

[java-triton]:https://github.com/openjdk/babylon/blob/code-reflection/cr-examples/triton

### Vector addition

In this section we shall present the [Java version][java-vector-addition] of the
Triton vector addition program, and it's Java code model.

[java-vector-addition]: https://github.com/openjdk/babylon/blob/code-reflection/cr-examples/triton/src/test/java/oracle/code/triton/TestAddKernel.java

[//]: # (@formatter:off)
```java
@Reflect
static void add_kernel2(Ptr x_ptr,  // *Pointer* to first input vector.
                        Ptr y_ptr,  // *Pointer* to second input vector.
                        Ptr output_ptr,  // *Pointer* to output vector.
                        int n_elements,  // Size of the vector.
                        @Constant int BLOCK_SIZE)  // Number of elements each program should process.
// NOTE: @Constant so it can be used as a shape value
{
    // There are multiple 'programs' processing different data. We identify which program
    // we are here:
    var pid = Triton.programId(0); // We use a 1D launch grid so axis is 0.
    // This program will process inputs that are offset from the initial data.
    // For instance, if you had a vector of length 256 and block_size of 64, the programs
    // would each access the elements [0:64, 64:128, 128:192, 192:256].
    // Note that offsets is a list of pointers:
    var block_start = pid * BLOCK_SIZE;
    var range = Triton.arange(0, BLOCK_SIZE);
    var offsets = Triton.add(block_start, range);
    // Create a mask to guard memory operations against out-of-bounds accesses.
    var mask = Triton.compare(offsets, n_elements, Triton.CompareKind.LessThan);
    // Load x and y from DRAM, masking out any extra elements in case the input is not a
    // multiple of the block size.
    var x = Triton.load(Triton.add(x_ptr, offsets), mask);
    var y = Triton.load(Triton.add(y_ptr, offsets), mask);
    var output = Triton.add(x, y);
    // Write x + y back to DRAM.
    Triton.store(Triton.add(output_ptr, offsets), output, mask);
}
```
[//]: # (@formatter:on)

The Java method, `add_kernel2`, is annotated with `@Reflect`. This ensures there
is a Java code model available and accessible under similar access
control rules as for its invocation.

The `Triton` [class][Triton-class] has static methods, such as `arange`, that
define the Triton functionality, similarly named as their Python equivalents.
There are many other similarities to the equivalent Python version. Vertically
very similar. Less so horizontally. An obvious and notable difference is Java's
lack of operator overloading, hence the additional `Triton` static methods such
as `add` and `compare`.

(Is it possible for Java to support operator overloading both for numeric
scalars and tensors? The author thinks so and asks the reader to be patient -
stay tuned for possible details at a future date.)

[Triton-class]: https://github.com/openjdk/babylon/blob/code-reflection/cr-examples/triton/src/main/java/oracle/code/triton/Triton.java

However, notice that no explicit broadcasting is required. The arithmetic
methods accept arguments that are instances of `Number`. The Triton `Tensor`
and `Ptr` classes extend from `Number`. With autoboxing it is possible to mix
instances of `Tensor` and boxed scalars e.g., as with the
expression `Triton.add(block_start, range)`

### Explaining the Java code model

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
Java `try` statement (instances of `ExtendedOps.JavaTryOp`). Or, as we shall see
later define operations that model Triton programs.

What does the code model of `add_kernel2` look like? We can obtain the code
model at runtime and serialize its in-memory form to a textual form.

```
func @loc="107:5:file:/.../TestAddKernel.java" @"add_kernel2"
(%0 : java.type:"oracle.code.triton.Ptr",
%1 : java.type:"oracle.code.triton.Ptr",
%2 : java.type:"oracle.code.triton.Ptr",
%3 : java.type:"int",
%4 : java.type:"int")java.type:"void" -> {
    %5 : Var<java.type:"oracle.code.triton.Ptr"> = var %0 @loc="107:5" @"x_ptr";
    %6 : Var<java.type:"oracle.code.triton.Ptr"> = var %1 @loc="107:5" @"y_ptr";
    %7 : Var<java.type:"oracle.code.triton.Ptr"> = var %2 @loc="107:5" @"output_ptr";
    %8 : Var<java.type:"int"> = var %3 @loc="107:5" @"n_elements";
    %9 : Var<java.type:"int"> = var %4 @loc="107:5" @"BLOCK_SIZE";
    %10 : java.type:"int" = constant @loc="143:36" @0;
    %11 : java.type:"int" = invoke %10 @loc="143:19" @java.ref:"oracle.code.triton.Triton::programId(int):int";
    %12 : Var<java.type:"int"> = var %11 @loc="143:9" @"pid";
    %13 : java.type:"int" = var.load %12 @loc="148:27";
    %14 : java.type:"int" = var.load %9 @loc="148:33";
    %15 : java.type:"int" = mul %13 %14 @loc="148:27";
    %16 : Var<java.type:"int"> = var %15 @loc="148:9" @"block_start";
    %17 : java.type:"int" = constant @loc="149:35" @0;
    %18 : java.type:"int" = var.load %9 @loc="149:38";
    %19 : java.type:"oracle.code.triton.Tensor" = invoke %17 %18 @loc="149:21" @java.ref:"oracle.code.triton.Triton::arange(int, int):oracle.code.triton.Tensor";
    %20 : Var<java.type:"oracle.code.triton.Tensor"> = var %19 @loc="149:9" @"range";
    %21 : java.type:"int" = var.load %16 @loc="150:34";
    %22 : java.type:"java.lang.Integer" = invoke %21 @loc="150:23" @java.ref:"java.lang.Integer::valueOf(int):java.lang.Integer";
    %23 : java.type:"oracle.code.triton.Tensor" = var.load %20 @loc="150:47";
    %24 : java.type:"oracle.code.triton.Tensor" = invoke %22 %23 @loc="150:23" @java.ref:"oracle.code.triton.Triton::add(java.lang.Number, java.lang.Number):oracle.code.triton.Tensor";
    %25 : Var<java.type:"oracle.code.triton.Tensor"> = var %24 @loc="150:9" @"offsets";
    %26 : java.type:"oracle.code.triton.Tensor" = var.load %25 @loc="152:35";
    %27 : java.type:"int" = var.load %8 @loc="152:44";
    %28 : java.type:"java.lang.Integer" = invoke %27 @loc="152:20" @java.ref:"java.lang.Integer::valueOf(int):java.lang.Integer";
    %29 : java.type:"oracle.code.triton.Triton$CompareKind" = field.load @loc="152:56" @java.ref:"oracle.code.triton.Triton$CompareKind::LessThan:oracle.code.triton.Triton$CompareKind";
    %30 : java.type:"oracle.code.triton.Tensor" = invoke %26 %28 %29 @loc="152:20" @java.ref:"oracle.code.triton.Triton::compare(java.lang.Number, java.lang.Number, oracle.code.triton.Triton$CompareKind):oracle.code.triton.Tensor";
    %31 : Var<java.type:"oracle.code.triton.Tensor"> = var %30 @loc="152:9" @"mask";
    %32 : java.type:"oracle.code.triton.Ptr" = var.load %5 @loc="155:40";
    %33 : java.type:"oracle.code.triton.Tensor" = var.load %25 @loc="155:47";
    %34 : java.type:"oracle.code.triton.Tensor" = invoke %32 %33 @loc="155:29" @java.ref:"oracle.code.triton.Triton::add(java.lang.Number, java.lang.Number):oracle.code.triton.Tensor";
    %35 : java.type:"oracle.code.triton.Tensor" = var.load %31 @loc="155:57";
    %36 : java.type:"oracle.code.triton.Tensor" = invoke %34 %35 @loc="155:17" @java.ref:"oracle.code.triton.Triton::load(oracle.code.triton.Tensor, oracle.code.triton.Tensor):oracle.code.triton.Tensor";
    %37 : Var<java.type:"oracle.code.triton.Tensor"> = var %36 @loc="155:9" @"x";
    %38 : java.type:"oracle.code.triton.Ptr" = var.load %6 @loc="156:40";
    %39 : java.type:"oracle.code.triton.Tensor" = var.load %25 @loc="156:47";
    %40 : java.type:"oracle.code.triton.Tensor" = invoke %38 %39 @loc="156:29" @java.ref:"oracle.code.triton.Triton::add(java.lang.Number, java.lang.Number):oracle.code.triton.Tensor";
    %41 : java.type:"oracle.code.triton.Tensor" = var.load %31 @loc="156:57";
    %42 : java.type:"oracle.code.triton.Tensor" = invoke %40 %41 @loc="156:17" @java.ref:"oracle.code.triton.Triton::load(oracle.code.triton.Tensor, oracle.code.triton.Tensor):oracle.code.triton.Tensor";
    %43 : Var<java.type:"oracle.code.triton.Tensor"> = var %42 @loc="156:9" @"y";
    %44 : java.type:"oracle.code.triton.Tensor" = var.load %37 @loc="157:33";
    %45 : java.type:"oracle.code.triton.Tensor" = var.load %43 @loc="157:36";
    %46 : java.type:"oracle.code.triton.Tensor" = invoke %44 %45 @loc="157:22" @java.ref:"oracle.code.triton.Triton::add(java.lang.Number, java.lang.Number):oracle.code.triton.Tensor";
    %47 : Var<java.type:"oracle.code.triton.Tensor"> = var %46 @loc="157:9" @"output";
    %48 : java.type:"oracle.code.triton.Ptr" = var.load %7 @loc="159:33";
    %49 : java.type:"oracle.code.triton.Tensor" = var.load %25 @loc="159:45";
    %50 : java.type:"oracle.code.triton.Tensor" = invoke %48 %49 @loc="159:22" @java.ref:"oracle.code.triton.Triton::add(java.lang.Number, java.lang.Number):oracle.code.triton.Tensor";
    %51 : java.type:"oracle.code.triton.Tensor" = var.load %47 @loc="159:55";
    %52 : java.type:"oracle.code.triton.Tensor" = var.load %31 @loc="159:63";
    invoke %50 %51 %52 @loc="159:9" @java.ref:"oracle.code.triton.Triton::store(oracle.code.triton.Tensor, oracle.code.triton.Tensor, oracle.code.triton.Tensor):void";
    return @loc="107:5";
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
form, all of which extend from the abstract class `java.incubator.code.Op`.

The entry block has four block parameters which model `add_kernel2`'s method
parameters. These parameters are used as operands of various operations. Many
operations produce operation results, e.g., `%15` the result of a multiplication
operation, that are used as operands of subsequent operations, and so on.
The `return` operation has a result, again like all other operations, but since
that result cannot be meaningfully used we don't present it.

Code models have the property of Static Single-Assignment (SSA). We refer to
variables that can only be assigned once as values (they are a bit like final
variables in Java) .e.g., value `%15` can never be modified. A variable
declaration is modeled as an operation that produces a value that holds a
value (a box), and access operations load or store to that box.

We can see how the operations model Java language constructs like method
declarations, variables (method parameters or local variables) and access of
variables, binary and unary mathematical operations, or method invocations (
e.g., to method `Triton::programId`).

We can also see that the general structure of the code model is very similar to
the Triton MLIR program presented earlier. The contents are naturally very
different, since this code model models a Java program. The compiler we will
describe will transform this code model into another code model with similar
structure and content as the Triton MLIR program.

### Analyzing the Java code model

Before we can transform the Java code model we need to analyze it, performing
type checking and attributing richer types to all the values declared in the
Java code model.

To do this we shall traverse the code model building a map of value to computed
type (an instance of `Map<j.l.r.code.Value, j.l.r.code.TypeElement>`), that
represents the result of attribution after traversal is complete. In effect the
traversal performs an _abstract interpretation_ of the code. For each operation
encountered we will perform some type-based computation based on its semantics.

We first have to initialize (or seed) the map with the method's parameters and
their richer types. Here is the test method for testing the vector addition
program. It provides the list of _code model types_ that are attributed to the
program's parameters.

[//]: # (@formatter:off)
```java
@TritonTestExtension.Kernel("add_kernel2")
@Test
public void test2(TritonTestData t) {
    List<TypeElement> argTypes = List.of(
            new PtrType(JavaType.FLOAT),
            new PtrType(JavaType.FLOAT),
            new PtrType(JavaType.FLOAT),
            JavaType.INT,
            new ConstantType(JavaType.INT, 64));

    t.test(argTypes);
}
```
[//]: # (@formatter:on)

This is conceptually similar to the signature shown earlier that was passed to
the (Python) Triton compiler. We have:

- instances of `PtrType` encapsulating the type of value it points to, that are
  attributed to values of `Ptr`
- instances of `ConstantType` encapsulating the constant type and its value,
  that are attributed to values that are constant
- (similarly, but not shown), we also have instances of `TensorType`
  encapsulating shape and element type, that are attributed to values
  of `Tensor`.

These classes model Triton types, instances are Triton code model types
extending from `TritonType` which in turn extends from `j.l.r.code.TypeElement`.
The class `JavaType` models Java types, and similarly extends
from `j.l.r.code.TypeElement`. Therefore, we can use pattern matching to operate
on specific kinds of code model types, or alternatively we can uniformly operate
on all code model types.

Triton types are not Java types i.e., they cannot be declared in Java programs
as the types of variables. However, notice that Triton types can conveniently
reuse other kinds of code model types for their own purposes e.g., a pointer to
32-bit signed integers, conveniently reusing the `JavaType.INT` code model type
that models the Java primitive `int` type.

When we encounter an operation we look up the attributed types of its operands
(values) that were previously computed, and from those types compute a type that
is attributed to the operation's result. And so on for subsequent operations.

We can lean into this concept and provide a higher-level abstraction for
invocation operations to methods on the `Triton` class and implement a "mirror"
class, `TritonTypeInterpreter`, that has methods of the same name with
parameters corresponding to the expected attributed types. When we encounter
such an invoke operation we obtain the attributed types from the operands using
the map and then, using reflection, invoke the method on the mirror class.

Here's the attribute implementation for the `Triton.arange` method:

```java
//                Tensor arange(@Constant int start, @Constant int end)
public static TensorType arange(ConstantType start, ConstantType end) {
    assert start.cType().equals(JavaType.INT);
    assert end.cType().equals(JavaType.INT);

    int startValue = (int) start.value();
    int endValue = (int) end.value();

    return new TensorType(JavaType.INT, List.of(endValue - startValue));
}
```

We expect that both arguments passed to `Triton.arange` are integer constant
types. From the constant parameters, `start` and `end`, we can construct and
return a `TensorType` of one dimension with the appropriate size, and whose
elements are 32-bit integers.

Here's the attribute implementation for the `Triton.load` method:

```java
//                Tensor load(Tensor ptr, Tensor mask)
public static TensorType load(TensorType ptr, TensorType mask) {
    checkTensorShape(ptr, mask);
    if (ptr.eType() instanceof PtrType eptr) {
        return new TensorType(eptr.rType(), ptr.shape());
    }

    throw new IllegalStateException();
}
```

In this case we know that `Triton.load` can only accept values that are tensors,
and so they are attributed tensor types. We first check that the tensor of
pointers is shape compatible with the mask (either its of the same shape as the
pointer or can be broadcast to that shape). Then we check that the tensor's
element type is a pointer type. If so, we construct and return a tensor type
whose shape is the same as the tensor of pointers, and whose element type is
type pointed to.

Below is a snippet of the Java code model presented above with comments showing
the mapping of values to attributed types.

```
    %16 : Var<java.type:"int"> = var %15 @loc="148:9" @"block_start";
 // %16 : Var<java.type.primitive<int>> -> int
    %17 : java.type:"int" = constant @loc="149:35" @0;
 // %17 : int -> constant<java.type.primitive<int>, c0>
    %18 : java.type:"int" = var.load %9 @loc="149:38";
 // %18 : int -> constant<java.type.primitive<int>, c64>
    %19 : java.type:"oracle.code.triton.Tensor" = invoke %17 %18 @loc="149:21"
            @java.ref:"oracle.code.triton.Triton::arange(int, int):oracle.code.triton.Tensor";
 // %19 : oracle.code.triton.Tensor -> tensor<x64, java.type.primitive<int>>
    %20 : Var<java.type:"oracle.code.triton.Tensor"> = var %19 @loc="149:9" @"range";
 // %20 : Var<java.type.class<oracle.code.triton.Tensor, java.type.primitive<void>>> -> tensor<x64, java.type.primitive<int>>
    %21 : java.type:"int" = var.load %16 @loc="150:34";
 // %21 : int -> int
    %22 : java.type:"java.lang.Integer" = invoke %21 @loc="150:23" @java.ref:"java.lang.Integer::valueOf(int):java.lang.Integer";
 // %22 : java.lang.Integer -> int
    %23 : java.type:"oracle.code.triton.Tensor" = var.load %20 @loc="150:47";
 // %23 : oracle.code.triton.Tensor -> tensor<x64, java.type.primitive<int>>
    %24 : java.type:"oracle.code.triton.Tensor" = invoke %22 %23 @loc="150:23"
            @java.ref:"oracle.code.triton.Triton::add(java.lang.Number, java.lang.Number):oracle.code.triton.Tensor";
 // %24 : oracle.code.triton.Tensor -> tensor<x64, java.type.primitive<int>>
```

### Transforming the Java code model

Once we have computed the map of values to attributed types we can transform the
Java code model to a Triton code model.

Code models are immutable. Code models can be produced by building, or by
transforming an existing code model. Transforming takes an input code model and
builds an output code model. For each input operation encountered in the input
code model we have the choice to add that operation to the builder of the output
code model (copying), to not add it (removal), or add new output operations
(replacement or addition). When an input operation is copied it's input result
is associated with the output result of the copied output operation. In other
cases it is possible to explicitly control this association. If a subsequently
encountered input operation uses that input result then the transformation can
obtain the output result from the input result.

In general code models are not bound to just modeling Java programs. It is
possible to define a set of operations and types that can be used to model
programs in a specific domain. The Triton code model will contain Triton
specific operations, and also those that correlate closely with dependent MLIR
dialects of Triton MLIR.

Values in a Triton code model can have Triton specific code model types. In
fact, we have already mentioned them in the prior section, they are types
attributed to values in the Java code model e.g., `TensorType` and `PtrType`. We
reuse them when building the Triton code model.

The goal here is to produce a code model that very similar to Triton MLIR. Note,
it is not a goal to compete with MLIR - we want to interoperate with it.

We can define a "mirror" of the `Triton` API for building operations associated
with invocations to that API called `TritonBuilderInterpreter` that has methods
of the same name (similar to `TritonTypeInterpreter` we discussed earlier). The
method parameters will be pairs of attributed type and input value, one pair for
each parameter and one pair for the return.

Here's the builder implementation for the `Triton.arange` method:

```java
//    Tensor arange(@Constant int start, @Constant int end)
public Value arange(TensorType rType, Op.Result r,
                    ConstantType startType, Value start,
                    ConstantType endType, Value end) {
    return block.op(TritonOps.makeRange(
            (int) startType.value(),
            (int) endType.value()));
}
```

We use the output block builder, `block`, to replace the invocation to
`Triton.arange` with the Triton make range operation, whose result type is the
same as `rType`. Note in this case we don't use `rType`, since the operation
reconstructs an equivalent instance. Instances of `TypeElement` are value-based,
so we could assert such equality e.g.,

[//]: # (@formatter:off)
```java
assert arangeOp.result().type().equals(rType);
```
[//]: # (@formatter:on)

Here's the builder implementation for the `Triton.load` method:

```java
//    Tensor load(Tensor ptr, Tensor mask)
public Value load(TensorType rType, Op.Result r,
                  TensorType ptrType, Value ptr,
                  TensorType maskType, Value mask) {
    broadcastConversionRight(ptrType, maskType, mask);
    return block.op(TritonOps.load(
            rType,
            block.context().getValue(ptr),
            block.context().getValue(mask)));
}
```

First, if necessary, we add a Triton operation that broadcasts the mask tensor
to a new mask tensor with same shape as the pointer tensor. Then we add the
Triton load operation whose result type is `rType`. The operands of the load
operation are the output values associated with the input values to the input
invocation operation. We must have previously encountered the input operations
whose results are those input values, hence there will be a mapping to the
output values. If a broadcast of the mask tensor is inserted then we will
re-associate input mask value with the new output mask value.

Here is the textual form of the Triton code model produced by compiling the
`add_kernel2`.

```
module ()java.type:"void" -> {
    tt.func @"add_kernel2_ptr<java.type.primitive<float>>_ptr<java.type.primitive<float>>_ptr<java.type.primitive<float>>_int_64_void" (
            %0 : ptr<java.type:"float">,
            %1 : ptr<java.type:"float">,
            %2 : ptr<java.type:"float">,
            %3 : java.type:"int")java.type:"void" -> {
        %4 : java.type:"int" = arith.constant @64;
        %5 : java.type:"int" = tt.get_program_id @0;
        %6 : java.type:"int" = arith.muli %5 %4;
        %7 : tensor<x64, java.type:"int"> = tt.make_range @start=0 @end=64;
        %8 : tensor<x64, java.type:"int"> = tt.splat %6;
        %9 : tensor<x64, java.type:"int"> = arith.addi %8 %7;
        %10 : tensor<x64, java.type:"int"> = tt.splat %3;
        %11 : tensor<x64, java.type:"boolean"> = arith.cmpi %9 %10 @"slt";
        %12 : tensor<x64, ptr<java.type:"float">> = tt.splat %0;
        %13 : tensor<x64, ptr<java.type:"float">> = tt.addptr %12 %9;
        %14 : tensor<x64, java.type:"float"> = tt.load %13 %11;
        %15 : tensor<x64, ptr<java.type:"float">> = tt.splat %1;
        %16 : tensor<x64, ptr<java.type:"float">> = tt.addptr %15 %9;
        %17 : tensor<x64, java.type:"float"> = tt.load %16 %11;
        %18 : tensor<x64, java.type:"float"> = arith.addf %14 %17;
        %19 : tensor<x64, ptr<java.type:"float">> = tt.splat %2;
        %20 : tensor<x64, ptr<java.type:"float">> = tt.addptr %19 %9;
        tt.store %20 %18 %11;
        tt.return;
    };
    unreachable;
};
```

We can see by design it is remarkably similar to the previously shown MLIR
version produced by the actual Triton compiler. In fact, it contains exactly the
same number of operations. Notice how the constant value for `BLOCK_SIZE`
has been folded into the operations and types.

### Further examples

The proof of concept Java Triton compiler has additional tests case that
implement Triton programs in Java for:

1. fused softmax [example][softmax], see [here][softmax-java] for the Java code;
   and
2. matrix multiply [example][matrix-multiply], see [here][matrix-multiply-java]
   for the Java code.

[softmax]: https://triton-lang.org/main/getting-started/tutorials/02-fused-softmax.html

[softmax-java]: https://github.com/openjdk/babylon/blob/code-reflection/cr-examples/triton/src/test/java/oracle/code/triton/TestSoftMax.java

[matrix-multiply]: https://triton-lang.org/main/getting-started/tutorials/03-matrix-multiplication.html

[matrix-multiply-java]: https://github.com/openjdk/babylon/blob/code-reflection/cr-examples/triton/src/test/java/oracle/code/triton/TestMatrix.java

The matrix multiply example is a compelling test case, with 2D tensors, various
forms of broadcast, tensor shape expansion, computations using 16-bit floats
expanding to 32-bits floats and back, and control flow. In the appendix we
describe aspects of this example in more detail.

## Appendix: Triton matrix multiply loop

Below we show snippets of the matrix multiply accumulating loop in Python and
Java.

Python:

[//]: # (@formatter:off)
```python
    # -----------------------------------------------------------
    # Iterate to compute a block of the C matrix.
    # We accumulate into a `[BLOCK_SIZE_M, BLOCK_SIZE_N]` block
    # of fp32 values for higher accuracy.
    # `accumulator` will be converted back to fp16 after the loop.
    accumulator = tl.zeros((BLOCK_SIZE_M, BLOCK_SIZE_N), dtype=tl.float32)
    for k in range(0, tl.cdiv(K, BLOCK_SIZE_K)):
        # Load the next block of A and B, generate a mask by checking the K dimension.
        # If it is out of bounds, set it to 0.
        a = tl.load(a_ptrs, mask=offs_k[None, :] < K - k * BLOCK_SIZE_K, other=0.0)
        b = tl.load(b_ptrs, mask=offs_k[:, None] < K - k * BLOCK_SIZE_K, other=0.0)
        # We accumulate along the K dimension.
        accumulator += tl.dot(a, b)
        # Advance the ptrs to the next K block.
        a_ptrs += BLOCK_SIZE_K * stride_ak
        b_ptrs += BLOCK_SIZE_K * stride_bk
    # You can fuse arbitrary activation functions here
    # while the accumulator is still in FP32!
    if ACTIVATION == "leaky_relu":
        accumulator = leaky_relu(accumulator)
    c = accumulator.to(tl.float16)
```
[//]: # (@formatter:on)

Java:

[//]: # (@formatter:off)
```java
        // -----------------------------------------------------------
        // Iterate to compute a block of the C matrix.
        // We accumulate into a `[BLOCK_SIZE_M, BLOCK_SIZE_N]` block
        // of fp32 values for higher accuracy.
        // `accumulator` will be converted back to fp16 after the loop.
        var accumulator = zeros(float.class, BLOCK_SIZE_M, BLOCK_SIZE_N);
        for (int k = 0; k < cdiv(K, BLOCK_SIZE_K); k++) {
            // Load the next block of A and B, generate a mask by checking the K dimension.
            // If it is out of bounds, set it to 0.
            var a = load(a_ptrs,
                    compare(expand(offs_k, 0), K - k * BLOCK_SIZE_K, LessThan));
            var b = load(b_ptrs,
                    compare(expand(offs_k, 1), K - k * BLOCK_SIZE_K, LessThan));
            // We accumulate along the K dimension.
            accumulator = add(accumulator, dot(a, b));
            // Advance the ptrs to the next K block.
            a_ptrs = add(a_ptrs, BLOCK_SIZE_K * stride_ak);
            b_ptrs = add(b_ptrs, BLOCK_SIZE_K * stride_bk);
        }

        // You can fuse arbitrary activation functions here
        // while the accumulator is still in FP32!
//        if (ACTIVATION) {
//            // ...
//        }
        var c = Triton.conv(Float16.class, accumulator);
```
[//]: # (@formatter:on)

The Java code is more verbose because Java does not support operator
overloading, including that for overloading array slicing to expand a tensor's
dimensions. (Note the Java code in this example statically imports
`Triton`'s static methods.)

The matrix multiply cleverly organizes the computation into groups of blocks,
ensuring efficient use of memory. The matrix multiplication loops over blocks
of `K`, loading tensors from matrices `A(M, K)` and `B(K, N)`, accumulating the
multiplication of those tensors, the final result of which is stored to
`C(M, N)`. The multiplication of the tensors is performed by the Triton "dot"
operation. The MLIR Triton compiler can compile this operation to instructions
leveraging mixed precision Tensor Cores.

The Python Triton compiler transforms the AST of the Python `for` loop to the
MLIR `scf.for` [operation][scf.for]. The `for` loop must be a counted loop with
a well-defined lower bound, upper bound, and step. Further, the compiler has to
identify variables updated within the loop that are declared outside of it.
These will become loop-carried variables, the final values of which are returned
by the operation when the loop terminates. In this case there are three such
variables, `accumulator`, `a_ptrs`, and `b_ptrs`.

[scf.for]: https://mlir.llvm.org/docs/Dialects/SCFDialect/#scffor-scfforop

The Java Triton compiler needs to perform a similar transformation. Here is an
abridged snippet of the Java code model showing the modelled `for` loop (the
complete snippet is presented below in a subsequent section).

```
java.for @loc="240:9"
    ()Var<java.type:"int"> -> {
        %148 : java.type:"int" = constant @loc="240:22" @0;
        %149 : Var<java.type:"int"> = var %148 @loc="240:14" @"k";
        yield %149 @loc="240:9";
    }
    (%150 : Var<java.type:"int">)java.type:"boolean" -> {
        %151 : java.type:"int" = var.load %150 @loc="240:25";
        %152 : java.type:"int" = var.load %22 @loc="240:34";
        %153 : java.type:"java.lang.Integer" = invoke %152 @loc="240:29" @java.ref:"java.lang.Integer::valueOf(int):java.lang.Integer";
        %154 : java.type:"int" = var.load %31 @loc="240:37";
        %155 : java.type:"java.lang.Integer" = invoke %154 @loc="240:29" @java.ref:"java.lang.Integer::valueOf(int):java.lang.Integer";
        %156 : java.type:"int" = invoke %153 %155 @loc="240:29" @java.ref:"oracle.code.triton.Triton::cdiv(java.lang.Number, java.lang.Number):int";
        %157 : java.type:"boolean" = lt %151 %156 @loc="240:25";
        yield %157 @loc="240:9";
    }
    (%158 : Var<java.type:"int">)java.type:"void" -> {
        %159 : java.type:"int" = var.load %158 @loc="240:52";
        %160 : java.type:"int" = constant @loc="240:52" @1;
        %161 : java.type:"int" = add %159 %160 @loc="240:52";
        var.store %158 %161 @loc="240:52";
        yield @loc="240:9";
    }
    (%162 : Var<java.type:"int">)java.type:"void" -> {
        ...
    };
```

We can clearly see that this operation contain four bodies, each of which
corresponds to a nested expression or statement as specified by the Java
Language Specification (see [here][jls-for]).

[jls-for]: https://docs.oracle.com/javase/specs/jls/se21/html/jls-14.html#jls-14.14.1

The Java Triton compiler checks if the for loop is a counted loop by analyzing
the operations in the first three bodies. If so then the compiler extracts and
transforms the operations associated with computing the bounds and step, and
identifies and transforms loop-carried variables. In the latter case we need to
identify all `var.store` operations to values of variables that are declared
outside the loop i.e., a `var.store` operation must be a descendant of its
associated `var` operation in the code model tree.

(Note, since this is a proof of concept the analysis is currently very basic,
more extensive capabilities would be useful as functionality in the code
reflection analysis package.)

The resulting transformed snippet is shown below.

```
%76 : java.type:"int" = arith.constant @value=0;
%77 : java.type:"int" = tt.call %17 @callee="cdiv_int_32_int";
%78 : java.type:"int" = arith.constant @value=1;
%79 : Tuple<tensor<x32, x64, java.type:"float">,
            tensor<x32, x32, ptr<java.type:"oracle.code.triton.Float16">>,
            tensor<x32, x64, ptr<java.type:"oracle.code.triton.Float16">>> =
       scf.for %76 %77 %78 %75 %63 %74
       (%80 : java.type:"int",
        %81 : tensor<x32, x64, java.type:"float">,
        %82 : tensor<x32, x32, ptr<java.type:"oracle.code.triton.Float16">>,
        %83 : tensor<x32, x64, ptr<java.type:"oracle.code.triton.Float16">>)
              Tuple<tensor<x32, x64, java.type:"float">,
                    tensor<x32, x32, ptr<java.type:"oracle.code.triton.Float16">>,
                    tensor<x32, x64, ptr<java.type:"oracle.code.triton.Float16">>> -> {
    ...
    scf.yield %102 %105 %108;
};
%109 : tensor<x32, x64, java.type:"float"> = tuple.load %79 @0;
%110 : tensor<x32, x32, ptr<java.type:"oracle.code.triton.Float16">> = tuple.load %79 @1;
%111 : tensor<x32, x64, ptr<java.type:"oracle.code.triton.Float16">> = tuple.load %79 @2;
```

We can see that the values `%76`, `%77` and `%78` correspond to the bounds and
step. They are hoisted out of the counted loop and passed as operands. The
values `%75`, `%63`, and `%74`, also passed as operands, correspond to the
initial values of the loop-carried variables (the `accumulator`, `a_ptr`
and `b_ptr`). Within the body of the loop operation there is a terminal
operation, `scf.yield`, that yields the updated values of the loop-carried
variables for the next iteration or loop's result.

The code model design does not support operations returning multiple results.
Instead, we model that capability using the code model `Tuple` type, that
declares how many components there are and each component's type. Hence, the
loop operation returns a `Tuple` with three component values corresponding to
the final values of the loop-carried variables, after which the tuple's
component values are unpacked.

The Triton code model snippet of Java Triton matrix multiply loop and the Triton
MLIR snippet of (Python) Triton matrix multiply loop are presented below in
subsequent sections (see the Java [test][matrix-multiply-java] for all details).

### Java code model snippet of Java Triton matrix multiply loop

```
java.for @loc="240:9"
    ()Var<java.type:"int"> -> {
        %148 : java.type:"int" = constant @loc="240:22" @0;
        %149 : Var<java.type:"int"> = var %148 @loc="240:14" @"k";
        yield %149 @loc="240:9";
    }
    (%150 : Var<java.type:"int">)java.type:"boolean" -> {
        %151 : java.type:"int" = var.load %150 @loc="240:25";
        %152 : java.type:"int" = var.load %22 @loc="240:34";
        %153 : java.type:"java.lang.Integer" = invoke %152 @loc="240:29" @java.ref:"java.lang.Integer::valueOf(int):java.lang.Integer";
        %154 : java.type:"int" = var.load %31 @loc="240:37";
        %155 : java.type:"java.lang.Integer" = invoke %154 @loc="240:29" @java.ref:"java.lang.Integer::valueOf(int):java.lang.Integer";
        %156 : java.type:"int" = invoke %153 %155 @loc="240:29" @java.ref:"oracle.code.triton.Triton::cdiv(java.lang.Number, java.lang.Number):int";
        %157 : java.type:"boolean" = lt %151 %156 @loc="240:25";
        yield %157 @loc="240:9";
    }
    (%158 : Var<java.type:"int">)java.type:"void" -> {
        %159 : java.type:"int" = var.load %158 @loc="240:52";
        %160 : java.type:"int" = constant @loc="240:52" @1;
        %161 : java.type:"int" = add %159 %160 @loc="240:52";
        var.store %158 %161 @loc="240:52";
        yield @loc="240:9";
    }
    (%162 : Var<java.type:"int">)java.type:"void" -> {
        %163 : java.type:"oracle.code.triton.Tensor" = var.load %126 @loc="243:26";
        %164 : java.type:"oracle.code.triton.Tensor" = var.load %110 @loc="244:36";
        %165 : java.type:"int" = constant @loc="244:44" @0;
        %166 : java.type:"oracle.code.triton.Tensor" = invoke %164 %165 @loc="244:29" @java.ref:"oracle.code.triton.Triton::expand(oracle.code.triton.Tensor, int):oracle.code.triton.Tensor";
        %167 : java.type:"int" = var.load %22 @loc="244:48";
        %168 : java.type:"int" = var.load %162 @loc="244:52";
        %169 : java.type:"int" = var.load %31 @loc="244:56";
        %170 : java.type:"int" = mul %168 %169 @loc="244:52";
        %171 : java.type:"int" = sub %167 %170 @loc="244:48";
        %172 : java.type:"java.lang.Integer" = invoke %171 @loc="244:21" @java.ref:"java.lang.Integer::valueOf(int):java.lang.Integer";
        %173 : java.type:"oracle.code.triton.Triton$CompareKind" = field.load @loc="244:70" @java.ref:"oracle.code.triton.Triton$CompareKind::LessThan:oracle.code.triton.Triton$CompareKind";
        %174 : java.type:"oracle.code.triton.Tensor" = invoke %166 %172 %173 @loc="244:21" @java.ref:"oracle.code.triton.Triton::compare(java.lang.Number, java.lang.Number, oracle.code.triton.Triton$CompareKind):oracle.code.triton.Tensor";
        %175 : java.type:"float" = constant @loc="244:81" @0.0f;
        %176 : java.type:"oracle.code.triton.Tensor" = invoke %163 %174 %175 @loc="243:21" @java.ref:"oracle.code.triton.Triton::load(oracle.code.triton.Tensor, oracle.code.triton.Tensor, float):oracle.code.triton.Tensor";
        %177 : Var<java.type:"oracle.code.triton.Tensor"> = var %176 @loc="243:13" @"a";
        %178 : java.type:"oracle.code.triton.Tensor" = var.load %142 @loc="245:26";
        %179 : java.type:"oracle.code.triton.Tensor" = var.load %110 @loc="246:36";
        %180 : java.type:"int" = constant @loc="246:44" @1;
        %181 : java.type:"oracle.code.triton.Tensor" = invoke %179 %180 @loc="246:29" @java.ref:"oracle.code.triton.Triton::expand(oracle.code.triton.Tensor, int):oracle.code.triton.Tensor";
        %182 : java.type:"int" = var.load %22 @loc="246:48";
        %183 : java.type:"int" = var.load %162 @loc="246:52";
        %184 : java.type:"int" = var.load %31 @loc="246:56";
        %185 : java.type:"int" = mul %183 %184 @loc="246:52";
        %186 : java.type:"int" = sub %182 %185 @loc="246:48";
        %187 : java.type:"java.lang.Integer" = invoke %186 @loc="246:21" @java.ref:"java.lang.Integer::valueOf(int):java.lang.Integer";
        %188 : java.type:"oracle.code.triton.Triton$CompareKind" = field.load @loc="246:70" @java.ref:"oracle.code.triton.Triton$CompareKind::LessThan:oracle.code.triton.Triton$CompareKind";
        %189 : java.type:"oracle.code.triton.Tensor" = invoke %181 %187 %188 @loc="246:21" @java.ref:"oracle.code.triton.Triton::compare(java.lang.Number, java.lang.Number, oracle.code.triton.Triton$CompareKind):oracle.code.triton.Tensor";
        %190 : java.type:"float" = constant @loc="246:81" @0.0f;
        %191 : java.type:"oracle.code.triton.Tensor" = invoke %178 %189 %190 @loc="245:21" @java.ref:"oracle.code.triton.Triton::load(oracle.code.triton.Tensor, oracle.code.triton.Tensor, float):oracle.code.triton.Tensor";
        %192 : Var<java.type:"oracle.code.triton.Tensor"> = var %191 @loc="245:13" @"b";
        %193 : java.type:"oracle.code.triton.Tensor" = var.load %147 @loc="248:31";
        %194 : java.type:"oracle.code.triton.Tensor" = var.load %177 @loc="248:48";
        %195 : java.type:"oracle.code.triton.Tensor" = var.load %192 @loc="248:51";
        %196 : java.type:"oracle.code.triton.Tensor" = invoke %194 %195 @loc="248:44" @java.ref:"oracle.code.triton.Triton::dot(oracle.code.triton.Tensor, oracle.code.triton.Tensor):oracle.code.triton.Tensor";
        %197 : java.type:"oracle.code.triton.Tensor" = invoke %193 %196 @loc="248:27" @java.ref:"oracle.code.triton.Triton::add(java.lang.Number, java.lang.Number):oracle.code.triton.Tensor";
        var.store %147 %197 @loc="248:13";
        %198 : java.type:"oracle.code.triton.Tensor" = var.load %126 @loc="250:26";
        %199 : java.type:"int" = var.load %31 @loc="250:34";
        %200 : java.type:"int" = var.load %24 @loc="250:49";
        %201 : java.type:"int" = mul %199 %200 @loc="250:34";
        %202 : java.type:"java.lang.Integer" = invoke %201 @loc="250:22" @java.ref:"java.lang.Integer::valueOf(int):java.lang.Integer";
        %203 : java.type:"oracle.code.triton.Tensor" = invoke %198 %202 @loc="250:22" @java.ref:"oracle.code.triton.Triton::add(java.lang.Number, java.lang.Number):oracle.code.triton.Tensor";
        var.store %126 %203 @loc="250:13";
        %204 : java.type:"oracle.code.triton.Tensor" = var.load %142 @loc="251:26";
        %205 : java.type:"int" = var.load %31 @loc="251:34";
        %206 : java.type:"int" = var.load %25 @loc="251:49";
        %207 : java.type:"int" = mul %205 %206 @loc="251:34";
        %208 : java.type:"java.lang.Integer" = invoke %207 @loc="251:22" @java.ref:"java.lang.Integer::valueOf(int):java.lang.Integer";
        %209 : java.type:"oracle.code.triton.Tensor" = invoke %204 %208 @loc="251:22" @java.ref:"oracle.code.triton.Triton::add(java.lang.Number, java.lang.Number):oracle.code.triton.Tensor";
        var.store %142 %209 @loc="251:13";
        java.continue @loc="240:9";
    };
```

### MLIR snippet of (Python) Triton matrix multiply loop

```
%47 = tt.call @"zeros____0cconstexpr_(constexpr_32_, constexpr_64_)__1cconstexpr_fp32_"() : () -> tensor<32x64xf32>
%48 = tt.call @cdiv__i32__1cconstexpr_32_(%arg5) : (i32) -> i32
%c0_i32 = arith.constant 0 : i32
%c1_i32 = arith.constant 1 : i32
%49 = arith.bitcast %c0_i32 : i32 to i32
%50 = arith.bitcast %48 : i32 to i32
%51 = arith.bitcast %c1_i32 : i32 to i32
%52 = llvm.mlir.undef : i32
%53:3 = scf.for %arg12 = %49 to %50 step %51 iter_args(%arg13 = %47, %arg14 = %35, %arg15 = %46) -> (tensor<32x64xf32>, tensor<32x32x!tt.ptr<f16, 1>>, tensor<32x64x!tt.ptr<f16, 1>>)  : i32 {
  %83 = tt.expand_dims %24 {axis = 0 : i32} : (tensor<32xi32>) -> tensor<1x32xi32>
  %c32_i32_3 = arith.constant 32 : i32
  %84 = arith.muli %arg12, %c32_i32_3 : i32
  %85 = arith.subi %arg5, %84 : i32
  %86 = tt.splat %85 : (i32) -> tensor<1x32xi32>
  %87 = arith.cmpi slt, %83, %86 : tensor<1x32xi32>
  %cst = arith.constant 0.000000e+00 : f32
  %88 = tt.broadcast %87 : (tensor<1x32xi1>) -> tensor<32x32xi1>
  %cst_4 = arith.constant dense<0.000000e+00> : tensor<32x32xf32>
  %89 = arith.truncf %cst_4 : tensor<32x32xf32> to tensor<32x32xf16>
  %90 = tt.load %arg14, %88, %89 {cache = 1 : i32, evict = 1 : i32, isVolatile = false} : tensor<32x32xf16>
  %91 = tt.expand_dims %24 {axis = 1 : i32} : (tensor<32xi32>) -> tensor<32x1xi32>
  %c32_i32_5 = arith.constant 32 : i32
  %92 = arith.muli %arg12, %c32_i32_5 : i32
  %93 = arith.subi %arg5, %92 : i32
  %94 = tt.splat %93 : (i32) -> tensor<32x1xi32>
  %95 = arith.cmpi slt, %91, %94 : tensor<32x1xi32>
  %cst_6 = arith.constant 0.000000e+00 : f32
  %96 = tt.broadcast %95 : (tensor<32x1xi1>) -> tensor<32x64xi1>
  %cst_7 = arith.constant dense<0.000000e+00> : tensor<32x64xf32>
  %97 = arith.truncf %cst_7 : tensor<32x64xf32> to tensor<32x64xf16>
  %98 = tt.load %arg15, %96, %97 {cache = 1 : i32, evict = 1 : i32, isVolatile = false} : tensor<32x64xf16>
  %cst_8 = arith.constant 0.000000e+00 : f32
  %cst_9 = arith.constant dense<0.000000e+00> : tensor<32x64xf32>
  %99 = tt.dot %90, %98, %cst_9 {allowTF32 = true, maxNumImpreciseAcc = 0 : i32} : tensor<32x32xf16> * tensor<32x64xf16> -> tensor<32x64xf32>
  %100 = arith.addf %arg13, %99 : tensor<32x64xf32>
  %c32_i32_10 = arith.constant 32 : i32
  %101 = arith.muli %arg7, %c32_i32_10 : i32
  %102 = tt.splat %101 : (i32) -> tensor<32x32xi32>
  %103 = tt.addptr %arg14, %102 : tensor<32x32x!tt.ptr<f16, 1>>, tensor<32x32xi32>
  %c32_i32_11 = arith.constant 32 : i32
  %104 = arith.muli %arg8, %c32_i32_11 : i32
  %105 = tt.splat %104 : (i32) -> tensor<32x64xi32>
  %106 = tt.addptr %arg15, %105 : tensor<32x64x!tt.ptr<f16, 1>>, tensor<32x64xi32>
  scf.yield %100, %103, %106 : tensor<32x64xf32>, tensor<32x32x!tt.ptr<f16, 1>>, tensor<32x64x!tt.ptr<f16, 1>>
}
%54 = arith.truncf %53#0 : tensor<32x64xf32> to tensor<32x64xf16>
```

### Triton code model snippet of Java Triton matrix multiply loop

```
%76 : java.type:"int" = arith.constant @value=0;
%77 : java.type:"int" = tt.call %17 @callee="cdiv_int_32_int";
%78 : java.type:"int" = arith.constant @value=1;
%79 : Tuple<tensor<x32, x64, java.type:"float">, tensor<x32, x32, ptr<java.type:"oracle.code.triton.Float16">>, tensor<x32, x64, ptr<java.type:"oracle.code.triton.Float16">>> = scf.for %76 %77 %78 %75 %63 %74 (%80 : java.type:"int", %81 : tensor<x32, x64, java.type:"float">, %82 : tensor<x32, x32, ptr<java.type:"oracle.code.triton.Float16">>, %83 : tensor<x32, x64, ptr<java.type:"oracle.code.triton.Float16">>)Tuple<tensor<x32, x64, java.type:"float">, tensor<x32, x32, ptr<java.type:"oracle.code.triton.Float16">>, tensor<x32, x64, ptr<java.type:"oracle.code.triton.Float16">>> -> {
    %84 : tensor<x1, x32, java.type:"int"> = tt.expand_dims %52 @axis=0;
    %85 : java.type:"int" = arith.muli %80 %26;
    %86 : java.type:"int" = arith.subi %17 %85;
    %87 : tensor<x1, x32, java.type:"int"> = tt.splat %86;
    %88 : tensor<x1, x32, java.type:"boolean"> = arith.cmpi %84 %87 @predicate="slt";
    %89 : tensor<x32, x32, java.type:"boolean"> = tt.broadcast %88;
    %90 : tensor<x32, x32, java.type:"oracle.code.triton.Float16"> = arith.constant @value=0.0f;
    %91 : tensor<x32, x32, java.type:"oracle.code.triton.Float16"> = tt.load %82 %89 %90;
    %92 : tensor<x32, x1, java.type:"int"> = tt.expand_dims %52 @axis=1;
    %93 : java.type:"int" = arith.muli %80 %26;
    %94 : java.type:"int" = arith.subi %17 %93;
    %95 : tensor<x32, x1, java.type:"int"> = tt.splat %94;
    %96 : tensor<x32, x1, java.type:"boolean"> = arith.cmpi %92 %95 @predicate="slt";
    %97 : tensor<x32, x64, java.type:"boolean"> = tt.broadcast %96;
    %98 : tensor<x32, x64, java.type:"oracle.code.triton.Float16"> = arith.constant @value=0.0f;
    %99 : tensor<x32, x64, java.type:"oracle.code.triton.Float16"> = tt.load %83 %97 %98;
    %100 : tensor<x32, x64, java.type:"float"> = arith.constant @value=0.0f;
    %101 : tensor<x32, x64, java.type:"float"> = tt.dot %91 %99 %100;
    %102 : tensor<x32, x64, java.type:"float"> = arith.addf %81 %101;
    %103 : java.type:"int" = arith.muli %26 %21;
    %104 : tensor<x32, x32, java.type:"int"> = tt.splat %103;
    %105 : tensor<x32, x32, ptr<java.type:"oracle.code.triton.Float16">> = tt.addptr %82 %104;
    %106 : java.type:"int" = arith.muli %26 %19;
    %107 : tensor<x32, x64, java.type:"int"> = tt.splat %106;
    %108 : tensor<x32, x64, ptr<java.type:"oracle.code.triton.Float16">> = tt.addptr %83 %107;
    scf.yield %102 %105 %108;
};
%109 : tensor<x32, x64, java.type:"float"> = tuple.load %79 @0;
%110 : tensor<x32, x32, ptr<java.type:"oracle.code.triton.Float16">> = tuple.load %79 @1;
%111 : tensor<x32, x64, ptr<java.type:"oracle.code.triton.Float16">> = tuple.load %79 @2;
%112 : tensor<x32, x64, java.type:"oracle.code.triton.Float16"> = arith.truncf %109;
```


