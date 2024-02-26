
# Project Babylon

Babylon's primary goal is to extend the reach of Java to foreign
programming models such as SQL, differentiable programming, machine
learning models, and GPUs. Babylon will achieve this with an enhancement
to reflective programming in Java, called code reflection.

This Project is sponsored by the [Core Libraries](/groups/core-libs) and
[Compiler](/groups/compiler) Groups.

## Summary

Focusing on the GPU example, suppose a Java developer wants to write a
GPU kernel in Java and execute it on a GPU. The developer's Java code
must, somehow, be analyzed and transformed into an executable GPU
kernel. A Java library could do that, but it requires access to the Java
code in symbolic form. Such access is, however, currently limited to the
use of non-standard APIs or to conventions at different points in the
program's life cycle (compile time or run time), and the symbolic forms
available (abstract syntax trees or bytecodes) are often ill-suited to
analysis and transformation.

Babylon will extend Java's reach to foreign programming models with an
enhancement to reflective programming in Java, called code reflection.
This will enable standard access, analysis, and transformation of Java
code in a suitable form. Support for a foreign programming model can
then be more easily implemented as a Java library.

Babylon will ensure that code reflection is fit for purpose by creating
a GPU programming model for Java that leverages code reflection and is
implemented as a Java library. To reduce the risk of bias we will also
explore, or encourage the exploration of, other programming models such
as SQL and differentiable programming, though we may do so less
thoroughly.

Code reflection consists of three parts:

  1.  The modeling of Java programs as code models, suitable for access,
      analysis, and transformation.

  2.  Enhancements to Java reflection, enabling access to code models at
      compile time and run time.

  3.  APIs to build, analyze, and transform code models.

## Documents

  - Articles
    - [Automatic differentiation of Java code using Code Reflection](articles/auto-diff) (February 2024)
    - [Emulating C# LINQ in Java using Code Reflection](articles/linq) (February 2024)
    - [Exploring Triton GPU programming for neural networks in Java](articles/triton)  (February 2024)
    
## Community

  - Mailing lists & news
    - [babylon-dev](https://mail.openjdk.org/mailman/listinfo/babylon-dev)
      ([archives](https://mail.openjdk.org/pipermail/babylon-dev/))
  - Talks
    - JVMLS 2023, Code Reflection,
      [slides](https://cr.openjdk.org/~psandoz/conferences/2023-JVMLS/Code-Reflection-JVMLS-23-08-07.pdf),
      [video](https://youtu.be/xbk9_6XA_IY)
    - JVMLS 2023, Java and GPU ... are we nearly there yet?,
      [video](https://youtu.be/lbKBu3lTftc)

## Repository

The project repository is accessible at
[github.com/openjdk/babylon](https://github.com/openjdk/babylon).
