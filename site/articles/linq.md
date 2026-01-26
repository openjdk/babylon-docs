# Emulating C# LINQ in Java using Code Reflection

#### Paul Sandoz {.author}

#### February 2024, Updated Dec 2025 {.date}

In this article we will explain how to emulate aspects of C#'s Language
Integrated Query ([LINQ](https://learn.microsoft.com/en-us/dotnet/csharp/linq/))
in Java using Code Reflection. Specifically, LINQ's capability to enable
translation of LINQ queries (C# expressions) to SQL statements.

Code Reflection is a Java platform feature being researched and developed under
OpenJDK Project [Babylon](https://openjdk.org/projects/babylon/).

We will introduce Code Reflection concepts and APIs as we go. The explanations
are neither exhaustive nor very detailed, they are designed to give the reader
an intuitive sense and understanding of Code Reflection and its capabilities.

## C# LINQ

C#'s guide to [LINQ](https://learn.microsoft.com/en-us/dotnet/csharp/linq/)
states:

> Language-Integrated Query (LINQ) is the name for a set of technologies based
> on the integration of query capabilities directly into the C# language.
> Traditionally, queries against data are expressed as simple strings without
> type checking at compile time or IntelliSense support. Furthermore, you have
> to learn a different query language for each type of data source: SQL
> databases, XML documents, various Web services, and so on. With LINQ, a query
> is a first-class language construct, just like classes, methods, and events.

Here's a simple example of a LINQ query to a database using the query syntax:

```
DB db = ...;

// Query for customers in London.
IQueryable<String> custQuery =
    from cust in db.Customers
    where cust.City == "London"
    select cust.ContactName;
```

The query syntax is syntactical sugar for ease of reading and writing LINQ
queries, but its really just a shortcut to method calls.

The class `DB` contains enclosed classes that model SQL tables, such
as `Customers` whose properties, `City` and `ContactName`, model rows in the
table.

We can write the same LINQ query using the method syntax.

```
DB db = ...;

// Query for customers in London.
IQueryable<String> custQuery =
    db.Customers
    .Where(cust => cust.City == "London")
    .Select(cust => cust.ContactName);
```

A LINQ query, an instance of `IQuerable`, has a C#
property [Expression][Q-Expr], whose value is a *symbolic representation* of the
LINQ query. In C# such symbolic representations are referred to as
*expression trees*. In the code above the expression tree for the
query `custQuery` is composed of the expression trees of the lambda expressions
passed to the `Where` and `Select` methods **and** the expression trees for the
invocation expressions to those methods.

[Q-Expr]: https://learn.microsoft.com/en-us/dotnet/api/system.linq.iqueryable.expression?view=net-8.0#system-linq-iqueryable-expression

A LINQ query's `Expression` property, an [*expression tree*][Expr-Tree], has a
pleasing characteristic. Evaluating the expression tree produces a new LINQ
query whose expression tree is equal to the one that was evaluated i.e., for the
mathematically inclined, let `Q` be the query, `Q.e` the query's expression,
and `E` the evaluation function (from an expression to a query),
then `E(Q. e) = Q`. However pleasing that is, it is not of much practical use.
Such expression trees are designed to be processed symbolically and
*transformed*. For example, transforming to a different programming domain such
as SQL, where expression trees of queries are translated to SQL statements.

[Expr-Tree]: https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/expression-trees/

We will focus on this aspect in Java, building a query whose equivalent
expression property is a *code model*, a symbolic representation of Java code,
that is produced by using features of Code Reflection.

## Emulating LINQ-like query expressions in Java

For the purposes of emulation we can express the same query in Java as follows.

```java
// A record modeling a table with three columns, one for each component
record Customer(String contactName, String phone, String city) {
}

QueryProvider qp = new linq.TestQueryProvider();

// Query all customers based in London, and return their names
@Reflect
QueryResult<Stream<String>> results = qp.query(Customer.class)
        .where(c -> c.city.equals("London"))
        .select(c -> c.contactName)
        .elements();
```

We use a `record`, `Customer`, to model an SQL table, where the record's
components corresponds to rows in that table. We can see that the Java code
looks very similar to the C# LINQ query using the method syntax.

In the following sections we will explain how this is implemented using Code
Reflection.

A proof of concept implementation is available as a [test][linq-test] located in
the Babylon repository. The implementation is far from complete.

[linq-test]:https://github.com/openjdk/babylon/tree/code-reflection/test/jdk/java/lang/reflect/code/linq

Let's expand the fluent query into individual statements, so we can see the
types.

```java
Queryable<Customer> allCustomers = qp.query(Customer.class);
@Reflect
Queryable<Customer> londonCustomers = allCustomers.where(c -> c.city.equals("London"));
@Reflect
Queryable<String> londonCustomerNames = londonCustomers.select(c -> c.contactName);
QueryResult<Stream<String>> results = londonCustomerNames.elements();
```

The first three method calls produce instances of `Queryable`, and the final
call produces an instance of `QueryResult`. Each instance will have a symbolic
representation, a code model, of the query. We shall see that later on.

First, we create a new query, which returns an instance
of `Queryable<Customer>`. From that we "filter" for customers located with in
the city of London with a call to `where`, accepting a lambda expression that
returns true if the customer's `city` component is equal to the string "London".
Then we "map" customers to their contact names with a call to `select`,
accepting a lambda expression that returns the customer's `contactName`
component. Finally, we produce a query result with a call to `elements`, which
informs how we consume the result, in this case as a stream of customer contact
names.

The signatures of the `where` and `select` methods are as follows.

```java
default Queryable<T> where(Predicate<T> f) { /* ... */ }

default <R> Queryable<R> select(Function<T, R> f) { /* ... */ }
```

The `@Reflect` annotation declared on the local variable assignment statements
ensures code models for the lambda expressions, present in the assignment's
initializer expression, will be built by `javac` and made accessible at runtime.

### Obtaining a code model

Let's first focus on obtaining the code model of a reflectable lambda expression.
We shall return to how code models for queries are composed and built in later
sections.

We can obtain the code model of the `Predicate` instance, checking if a
customer is located in the city of London, as follows.

[//]: # (@formatter:off)
```java
@Reflect
Predicate<Customer> p = c -> c.city.equals("London");
Optional<JavaOp.LambdaOp> q = Op.ofLambda(p);
JavaOp.LambdaOp pcm = q.orElseThrow().op();
```
[//]: # (@formatter:on)

We call the `Op::ofLambda` method to obtain an optional instance of `Quoted`
from which we obtain the code model with a call to `Quoted::op`. In this case
the lambda expression does not capture any values, but if it did we could obtain
those values from the quoted instance.

The code model of `p` is represented as an instance of `JavaOp.LambdaOp`
corresponding to a *lambda expression* operation that models a lambda
expression. Since quoting is potentially not limited to just the quoting of
lambda expressions we first obtain the code model as an instance
of `java.incubator.code.Op`, the top-level type for all operations, from
which we then down cast.

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
operational semantics *declare* a function (instances of `CoreOp.FuncOp`),
model a Java lambda expression (instances of `JavaOp.LambdaOp`), or model a
Java `try` statement (instances of `JavaOp.TryOp`).

What does the code model of `p` look like? We can serialize its in-memory
form (the instance of `JavaOp.LambdaOp`) to a textual form.

```java
System.out.println(pcm.toText());
```

Which prints the following text.

```text
%0 : java.type:"java.util.function.Predicate<linq.TestLinq$Customer>" = lambda @lambda.isReflectable=true
(%1 : java.type:"linq.TestLinq$Customer")java.type:"boolean" -> {
    %2 : Var<java.type:"linq.TestLinq$Customer"> = var %1 @"c";
    %3 : java.type:"linq.TestLinq$Customer" = var.load %2;
    %4 : java.type:"java.lang.String" = field.load %3 @java.ref:"linq.TestLinq$Customer::city:java.lang.String";
    %5 : java.type:"java.lang.String" = constant @"London";
    %6 : java.type:"boolean" = invoke %4 %5 @java.ref:"java.lang.String::equals(java.lang.Object):boolean";
    return %6;
};
```

The textual form shows the code model's root is a lambda expression (`lambda`)
operation. The lambda expression operation has an operation result, like all
other operations, but since it's the root of the tree there is no need to
present it.

The lambda-like expression represents the fusion of the lambda expression
operation's single body and the body's first and only block, called the entry
block. Then there is a sequence of operations in the entry block. For each
operation there is an instance of a corresponding class present in the in-memory
form, all of which extend from the abstract class `java.incubator.code.Op`.

The entry block has one block parameters, `%1` (corresponding to `c`), described
by a type of `linq.TestLinq$Customer`, which models `p`'s parameter. This
parameter is used as an operand of another operation. Many operations produce
operation results, e.g., `%4` the result of a field load operation, that are
used as operands of subsequent operations, and so on. The `return`
operation has a result, again like all other operations, but since that result
cannot be meaningfully used we don't present it.

Code models have the property of Static Single-Assignment (SSA). We refer to
variables that can only be assigned once as values (they are a bit like final
variables in Java) .e.g., value `%4` can never be modified. A variable
declaration is modeled as an operation that produces a value that holds a
value (a box), and access operations load or store to that box.

(Some readers may be thinking this looks very similar
to [MLIR](https://mlir.llvm.org/) and that is by design.)

We can see how the operations model Java language constructs like lambda
expressions, variables (method parameters or local variables) and access of
variables, field access, or method invocations (e.g., to
method `String::equals`).

The field load and invoke operations are referred to as reflective operations.
They declare *references* (an operation attribute prefixed with `@`) that can
be used to construct method handles, or to generate bytecode instructions and
their constant pool entries.

## Code models of queries

For each queryable and query result instance we can obtain its code model and
print out the textual form.

The initial queryable is as follows.

```
Queryable<Customer> allCustomers = qp.query(Customer.class);
System.out.println(allCustomers.expression().toText());
```

```text
func @"query" (%0 : java.type:"linq.Queryable<linq.TestLinq$Customer>")java.type:"linq.Queryable<linq.TestLinq$Customer>" -> {
    return %0;
};
```

The initial queryable instance has a code model whose root is a *function
declaration* operation. A function declaration operation is similarly structured
as a lambda expression operation we have previously described. This code model
represents an identity function, returning the parameter `%0`. The parameter's
type is `linq.Queryable<linq.TestLinq$Customer>` that encapsulates the
`Customer` type, so we know what table we are querying.

The second queryable is as follows.

```
@Reflect
Queryable<Customer> londonCustomers = allCustomers.where(c -> c.city.equals("London"));
System.out.println(londonCustomers.expression().toText());
```

```text
func @"query" (%0 : java.type:"linq.Queryable<linq.TestLinq$Customer>")java.type:"linq.Queryable<linq.TestLinq$Customer>" -> {
    %1 : java.type:"java.util.function.Predicate<linq.TestLinq$Customer>" = lambda @lambda.isReflectable=true (%2 : java.type:"linq.TestLinq$Customer")java.type:"boolean" -> {
        %3 : Var<java.type:"linq.TestLinq$Customer"> = var %2 @"c";
        %4 : java.type:"linq.TestLinq$Customer" = var.load %3;
        %5 : java.type:"java.lang.String" = field.load %4 @java.ref:"linq.TestLinq$Customer::city:java.lang.String";
        %6 : java.type:"java.lang.String" = constant @"London";
        %7 : java.type:"boolean" = invoke %5 %6 @java.ref:"java.lang.String::equals(java.lang.Object):boolean";
        return %7;
    };
    %8 : java.type:"linq.Queryable<linq.TestLinq$Customer>" = invoke %0 %1 @java.ref:"linq.Queryable::where(java.util.function.Predicate):linq.Queryable";
    return %8;
};
```

The code model of the second queryable contains the lambda expression whose
result is passed to an invocation of the `where` method, whose result is
returned.

The third queryable is as follows.

```
@Reflect
Queryable<String> londonCustomerNames = londonCustomers.select(c -> c.contactName);
System.out.println(londonCustomerNames.expression().toText());
```

```text
func @"query" (%0 : java.type:"linq.Queryable<linq.TestLinq$Customer>")java.type:"linq.Queryable<java.lang.String>" -> {
    %1 : java.type:"java.util.function.Predicate<linq.TestLinq$Customer>" = lambda @lambda.isReflectable=true (%2 : java.type:"linq.TestLinq$Customer")java.type:"boolean" -> {
        %3 : Var<java.type:"linq.TestLinq$Customer"> = var %2 @"c";
        %4 : java.type:"linq.TestLinq$Customer" = var.load %3;
        %5 : java.type:"java.lang.String" = field.load %4 @java.ref:"linq.TestLinq$Customer::city:java.lang.String";
        %6 : java.type:"java.lang.String" = constant @"London";
        %7 : java.type:"boolean" = invoke %5 %6 @java.ref:"java.lang.String::equals(java.lang.Object):boolean";
        return %7;
    };
    %8 : java.type:"linq.Queryable<linq.TestLinq$Customer>" = invoke %0 %1 @java.ref:"linq.Queryable::where(java.util.function.Predicate):linq.Queryable";
    %9 : java.type:"java.util.function.Function<linq.TestLinq$Customer, java.lang.String>" = lambda @lambda.isReflectable=true (%10 : java.type:"linq.TestLinq$Customer")java.type:"java.lang.String" -> {
        %11 : Var<java.type:"linq.TestLinq$Customer"> = var %10 @"c";
        %12 : java.type:"linq.TestLinq$Customer" = var.load %11;
        %13 : java.type:"java.lang.String" = field.load %12 @java.ref:"linq.TestLinq$Customer::contactName:java.lang.String";
        return %13;
    };
    %14 : java.type:"linq.Queryable<java.lang.String>" = invoke %8 %9 @java.ref:"linq.Queryable::select(java.util.function.Function):linq.Queryable";
    return %14;
};
```

The code model of the third queryable contains the lambda expression whose
result is passed to an invocation of the `select` method, whose result is
returned.

We can now see a pattern emerge, a queryable code model *reifies* the steps
taken to build the query.

The query result is as follows.

```
QueryResult<Stream<String>> results = londonCustomerNames.elements();
System.out.println(results.expression().toText());
```

```text
func @"queryResult" (%0 : java.type:"linq.Queryable<linq.TestLinq$Customer>")java.type:"linq.QueryResult<java.util.stream.Stream<java.lang.String>>" -> {
    %1 : java.type:"java.util.function.Predicate<linq.TestLinq$Customer>" = lambda @lambda.isReflectable=true (%2 : java.type:"linq.TestLinq$Customer")java.type:"boolean" -> {
        %3 : Var<java.type:"linq.TestLinq$Customer"> = var %2 @"c";
        %4 : java.type:"linq.TestLinq$Customer" = var.load %3;
        %5 : java.type:"java.lang.String" = field.load %4 @java.ref:"linq.TestLinq$Customer::city:java.lang.String";
        %6 : java.type:"java.lang.String" = constant @"London";
        %7 : java.type:"boolean" = invoke %5 %6 @java.ref:"java.lang.String::equals(java.lang.Object):boolean";
        return %7;
    };
    %8 : java.type:"linq.Queryable<linq.TestLinq$Customer>" = invoke %0 %1 @java.ref:"linq.Queryable::where(java.util.function.Predicate):linq.Queryable";
    %9 : java.type:"java.util.function.Function<linq.TestLinq$Customer, java.lang.String>" = lambda @lambda.isReflectable=true (%10 : java.type:"linq.TestLinq$Customer")java.type:"java.lang.String" -> {
        %11 : Var<java.type:"linq.TestLinq$Customer"> = var %10 @"c";
        %12 : java.type:"linq.TestLinq$Customer" = var.load %11;
        %13 : java.type:"java.lang.String" = field.load %12 @java.ref:"linq.TestLinq$Customer::contactName:java.lang.String";
        return %13;
    };
    %14 : java.type:"linq.Queryable<java.lang.String>" = invoke %8 %9 @java.ref:"linq.Queryable::select(java.util.function.Function):linq.Queryable";
    %15 : java.type:"linq.QueryResult<java.util.stream.Stream<java.lang.String>>" = invoke %14 @java.ref:"linq.Queryable::elements():linq.QueryResult";
    return %15;
};
```

Finally, the code model of the query result contains the invocation of
the `elements` method, whose result is returned.

The prior code model will be equivalent (in terms of program meaning) to the
code model of the following method.

```java

@Reflect
static QueryResult<Stream<String>> queryResult(Queryable<Customer> q) {
    return q.where(c -> c.city.equals("London"))
            .select(c -> c.contactName)
            .elements();
}
```

If we interpret `results`'s code model, passing in an initial queryable
instance, then it will naturally produce a queryable instance that has the same
code model. (Alternatively we could compile the code model and execute it.)

```
QueryResult<Stream<String>> resultsIterpreted = (QueryResult<Stream<String>>) Interpreter.invoke(MethodHandles.lookup(),
        results.expression(), qp.query(Customer.class));
System.out.println(resultsIterpreted.expression().toText());

assert results.expression().toText().equals(resultsIterpreted.expression().toText());
```

Clearly that is not of much practical use. The code model is designed to be
*transformed*. This code model contains sufficient symbolic information for it
to be transformed to an SQL query.

## Building code models for queries

Code models are immutable. Code models can be produced by building, or by
transforming an existing code model. Transforming takes an input code model and
builds an output code model. For each input operation encountered in the input
code model we have the choice to add that operation to the builder of the output
code model (copying), to not add it (removal), or add new output operations
(replacement or addition).

The implementations of `Queryable` methods, `where`, `select`, and `elements`,
transform a prior code model to a new code model with the additional operations.

Here is the implementation of the `where` method.

```text
@SuppressWarnings("unchecked")
default Queryable<T> where(Predicate<T> f) {
    JavaOp.LambdaOp l = Op.ofLambda(f).orElseThrow().op();
    return (Queryable<T>) insertQuery(elementType(), "where", l);
}
```

We obtain the code model for the lambda expression, as previously shown, and
pass that along with element type, the name of the method to the
method `insertQuery`, whose result is returned.

The method `insertQuery` performs the transformation.

```java
private Queryable<?> insertQuery(JavaType elementType, String methodName, JavaOp.LambdaOp lambdaOp) {
    // Copy function expression, replacing return operation
    FuncOp queryExpression = expression();
    JavaType queryableType = parameterized(Queryable.TYPE, elementType);
    FuncOp nextQueryExpression = func("query",
            functionType(queryableType, queryExpression.invokableType().parameterTypes()))
            .body(b -> Inliner.inline(b, queryExpression, b.parameters(), (block, query) -> {
                Result fi = block.op(lambdaOp);

                MethodRef md = method(Queryable.TYPE, methodName,
                        functionType(Queryable.TYPE, ((ClassType) lambdaOp.functionalInterface()).rawType()));
                Result queryable = block.op(JavaOp.invoke(queryableType, md, query, fi));

                block.op(return_(queryable));
            }));

    return provider().createQuery(elementType, nextQueryExpression);
}
```

We start by building a new function operation with the static factory method
`CoreOp.func`. A method reference is constructed that describes the function's
parameters and return type. Then we call the `body` method to build the function
operation's body. The implementation of `body` calls the lambda expression with
function body's *entry block builder*, from which we can add operations.

In this case we are going to *inline* the prior input code model (a previously
generated function declaration operation) into the output code model we are
building. Inlining transforms `queryExpression` by copying the contents of the
input function declaration operation's body *and* replacing the input return
operation with what is built by the lambda expression passed to the `inline`
method.

We replace the input return operation with a copy of the lambda expression
operation, an invocation to the queryable method, and a return the invocation's
result. We can freely copy the lambda expression operation as long as it does
not capture.

Because we are inlining we also need to map the queryExpression function's
parameters to the output functions parameters (specifically in both cases the
block parameters of the entry block of the body of the function operation), so
we also pass the latter to the `inline` method which takes care of the mapping.

Finally, we create a new instance of `Queryable` with the output code model.