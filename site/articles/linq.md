# Emulating C# LINQ in Java using Code Reflection
#### Paul Sandoz {.author}
#### February 2024 {.date}

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

// Find all customers based in London, and return their names
QueryResult<Stream<String>> results = qp.newQuery(Customer.class)
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
Queryable<Customer> londonCustomers = allCustomers.where(c -> c.city.equals("London"));
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
default Queryable<T> where(QuotablePredicate<T> f) { /* ... */ }

default <R> Queryable<R> select(QuotableFunction<T, R> f) { /* ... */ }
```

`QuotablePredicate` and `QuotableFunction` are functional interfaces that extend
from `Predicate` and `Function` and the Code Reflection
interface `java.lang.reflect.code.Quotable`.

Here is the declaration of `Quotable`.

```java

@FunctionalInterface
public interface QuotablePredicate<T> extends Quotable, Predicate<T> {
}
```

When a lambda expression is targeted to a quotable functional interface a code
model for the lambda expression will be built by the source compiler and made
accessible at runtime via the quotable instance. In a similar manner to how we
can obtain serializable lambda expressions (using `Serializable`) that can be
serialized, we can obtain *quotable* lambda expressions whose code model can be
obtained.

### Obtaining a code model

Let's first focus on obtaining the code model of a quotable lambda expression.
We shall return to how code models for queries are composed and built in later
sections.

We can obtain the code model of the `QuotablePredicate` instance, checking if a
customer is located in the city of London, as follows.

```java
QuotablePredicate<Customer> qp = c -> c.city.equals("London");
Quoted q = qp.quoted();
Op op = q.op();
CoreOps.LambdaOp qpcm = (CoreOps.LambdaOp) op;
```

We call the `Quotable::quoted` method to obtain an instance of `Quoted` from
which we obtain the code model with a call to `Quoted::op`. In this case the
lambda expression does not capture any values, but if it did we could obtain
those values from the quoted instance.

The code model of `qp` is represented as an instance of `CoreOps.LambdaOp`
corresponding to a *lambda expression* operation that models a lambda
expression. Since quoting is potentially not limited to just the quoting of
lambda expressions we first obtain the code model as an instance
of `java.lang.reflect.code.Op`, the top-level type for all operations, from
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
operational semantics *declare* a function (instances of `CoreOps.FuncOp`),
model a Java lambda expression (instances of `CoreOps.LambdaOp`), or model a
Java `try` statement (instances of `ExtendedOps.JavaTryOp`).

What does the code model of `qp` look like? We can serialize its in-memory
form (the instance of `CoreOps.LambdaOp`) to a textual form.

```java
System.out.println(qpcm.toText());
```

Which prints the following text.

```text
lambda (%0 : linq.TestLinq$Customer)boolean -> {
    %1 : Var<linq.TestLinq$Customer> = var %0 @"c";
    %2 : linq.TestLinq$Customer = var.load %1;
    %3 : java.lang.String = field.load %2 @"linq.TestLinq$Customer::city()java.lang.String";
    %4 : java.lang.String = constant @"London";
    %5 : boolean = invoke %3 %4 @"java.lang.String::equals(java.lang.Object)boolean";
    return %5;
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
form, all of which extend from the abstract class `java.lang.reflect.code.Op`.

The entry block has one block parameters, `%0` (corresponding to `c`), described
by a type of `linq.TestLinq$Customer`, which models `qp`'s parameter. This
parameter is used as an operand of another operation. Many operations produce
operation results, e.g., `%3` the result of a field load operation, that are
used as operands of subsequent operations, and so on. The `return`
operation has a result, again like all other operations, but since that result
cannot be meaningfully used we don't present it.

Code models have the property of Static Single-Assignment (SSA). We refer to
variables that can only be assigned once as values (they are a bit like final
variables in Java) .e.g., value `%3` can never be modified. A variable
declaration is modeled as an operation that produces a value that holds a
value (a box), and access operations load or store to that box.

(Some readers may be thinking this looks very similar
to [MLIR](https://mlir.llvm.org/) and that is by design.)

We can see how the operations model Java language constructs like lambda
expressions, variables (method parameters or local variables) and access of
variables, field access, or method invocations (e.g., to
method `String::equals`).

The field load and invoke operations are referred to as reflective operations.
They declare *descriptors* (an operation attribute prefixed with `@`) that can
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
func @"query" (%0 : linq.Queryable<linq.TestLinq$Customer>)linq.Queryable<linq.TestLinq$Customer> -> {
    return %0;
};
```

The initial queryable instance has a code model whose root is a *function
declaration* operation. A function declaration operation is similarly structured
as a lambda expression operation we have previously described. This code model
represents an identity function, returning the parameter `%0`. The parameter's
type description is `linq.Queryable<linq.TestLinq$Customer>` that encapsulates
the `Customer` type, so we know what table we are querying.

```
Queryable<Customer> londonCustomers = allCustomers.where(c -> c.city.equals("London"));
System.out.println(londonCustomers.expression().toText());
```

The second queryable is as follows.

```text
func @"query" (%0 : linq.Queryable<linq.TestLinq$Customer>)linq.Queryable<linq.TestLinq$Customer> -> {
    %1 : linq.QuotablePredicate<linq.TestLinq$Customer> = lambda (%2 : linq.TestLinq$Customer)boolean -> {
        %3 : Var<linq.TestLinq$Customer> = var %2 @"c";
        %4 : linq.TestLinq$Customer = var.load %3;
        %5 : java.lang.String = field.load %4 @"linq.TestLinq$Customer::city()java.lang.String";
        %6 : java.lang.String = constant @"London";
        %7 : boolean = invoke %5 %6 @"java.lang.String::equals(java.lang.Object)boolean";
        return %7;
    };
    %8 : linq.Queryable<linq.TestLinq$Customer> = invoke %0 %1 @"linq.Queryable::where(linq.QuotablePredicate)linq.Queryable";
    return %8;
};
```

The code model of the second queryable contains the lambda expression whose
result is passed to an invocation of the `where` method, whose result is
returned.

The third queryable is as follows.

```
Queryable<String> londonCustomerNames = londonCustomers.select(c -> c.contactName);
System.out.println(londonCustomerNames.expression().toText());
```

```text
func @"query" (%0 : linq.Queryable<linq.TestLinq$Customer>)linq.Queryable<java.lang.String> -> {
    %1 : linq.QuotablePredicate<linq.TestLinq$Customer> = lambda (%2 : linq.TestLinq$Customer)boolean -> {
        %3 : Var<linq.TestLinq$Customer> = var %2 @"c";
        %4 : linq.TestLinq$Customer = var.load %3;
        %5 : java.lang.String = field.load %4 @"linq.TestLinq$Customer::city()java.lang.String";
        %6 : java.lang.String = constant @"London";
        %7 : boolean = invoke %5 %6 @"java.lang.String::equals(java.lang.Object)boolean";
        return %7;
    };
    %8 : linq.Queryable<linq.TestLinq$Customer> = invoke %0 %1 @"linq.Queryable::where(linq.QuotablePredicate)linq.Queryable";
    %9 : linq.QuotableFunction<linq.TestLinq$Customer, java.lang.String> = lambda (%10 : linq.TestLinq$Customer)java.lang.String -> {
        %11 : Var<linq.TestLinq$Customer> = var %10 @"c";
        %12 : linq.TestLinq$Customer = var.load %11;
        %13 : java.lang.String = field.load %12 @"linq.TestLinq$Customer::contactName()java.lang.String";
        return %13;
    };
    %14 : linq.Queryable<java.lang.String> = invoke %8 %9 @"linq.Queryable::select(linq.QuotableFunction)linq.Queryable";
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
func @"queryResult" (%0 : linq.Queryable<linq.TestLinq$Customer>)linq.QueryResult<java.util.stream.Stream<java.lang.String>> -> {
    %1 : linq.QuotablePredicate<linq.TestLinq$Customer> = lambda (%2 : linq.TestLinq$Customer)boolean -> {
        %3 : Var<linq.TestLinq$Customer> = var %2 @"c";
        %4 : linq.TestLinq$Customer = var.load %3;
        %5 : java.lang.String = field.load %4 @"linq.TestLinq$Customer::city()java.lang.String";
        %6 : java.lang.String = constant @"London";
        %7 : boolean = invoke %5 %6 @"java.lang.String::equals(java.lang.Object)boolean";
        return %7;
    };
    %8 : linq.Queryable<linq.TestLinq$Customer> = invoke %0 %1 @"linq.Queryable::where(linq.QuotablePredicate)linq.Queryable";
    %9 : linq.QuotableFunction<linq.TestLinq$Customer, java.lang.String> = lambda (%10 : linq.TestLinq$Customer)java.lang.String -> {
        %11 : Var<linq.TestLinq$Customer> = var %10 @"c";
        %12 : linq.TestLinq$Customer = var.load %11;
        %13 : java.lang.String = field.load %12 @"linq.TestLinq$Customer::contactName()java.lang.String";
        return %13;
    };
    %14 : linq.Queryable<java.lang.String> = invoke %8 %9 @"linq.Queryable::select(linq.QuotableFunction)linq.Queryable";
    %15 : linq.QueryResult<java.util.stream.Stream<java.lang.String>> = invoke %14 @"linq.Queryable::elements()linq.QueryResult";
    return %15;
};
```

Finally, the code model of the query result contains the invocation of
the `elements` method, whose result is returned.

The prior code model will be equivalent (in terms of program meaning) to the
code model of the following method.

```java

@CodeReflection
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
default Queryable<T> where(QuotablePredicate<T> f) {
    CoreOps.LambdaOp l = (CoreOps.LambdaOp) f.quoted().op();
    return (Queryable<T>) insertQuery(elementType(), "where", l);
}
```

We obtain the code model for the lambda expression, as previously shown, and
pass that along with element type, the name of the method to the
method `insertQuery`, whose result is returned.

The method `insertQuery` performs the transformation.

```java
private Queryable<?> insertQuery(JavaType elementType, String methodName, LambdaOp lambdaOp) {
    // Copy function expression, replacing return operation
    FuncOp queryExpression = expression();
    JavaType queryableType = type(Queryable.TYPE, elementType);
    FuncOp nextQueryExpression = func("query",
            methodType(queryableType, queryExpression.funcDescriptor().parameters()))
            .body(b -> b.inline(queryExpression, b.parameters(), (block, query) -> {
                Op.Result fi = block.op(lambdaOp);

                MethodDesc md = method(Queryable.TYPE, methodName,
                        methodType(Queryable.TYPE, ((JavaType) lambdaOp.functionalInterface()).rawType()));
                Op.Result queryable = block.op(invoke(queryableType, md, query, fi));

                block.op(_return(queryable));
            }));

    return provider().createQuery(elementType, nextQueryExpression);
}
```

We start by building a new function declaration operation with the static
factory method `CoreOps.func`. A method description is constructed that
describes the function's parameters and return type. Then we call the `body`
method to build the function declaration operation's body. The implementation
of `body` calls the lambda expression with function body's *entry block
builder*, from which we can add operations.

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

Because we are inlining we also need to map the input function's parameters to
the output functions parameters (specifically in both cases the block parameters
of the entry block of the body of the function declaration operation), so we
also pass the latter to the `inline` method which takes care of the mapping.

Finally, we create a new instance of `Queryable` with the output code model.