# Type Classes and Polymorphism

[Type classes](https://en.wikipedia.org/wiki/Type_class) is a programming-language feature that exists in many programming languages (sometimes under the name _traits_, as in [Rust](https://doc.rust-lang.org/1.8.0/book/traits.html) or [Scala](https://docs.scala-lang.org/tour/traits.html)), and provides [ad-hoc polymorphism](https://en.wikipedia.org/wiki/Ad_hoc_polymorphism).

A type-class represents a contract, defined as a set of functions named _methods_. A type can be defined as an _instance_ of a type-class by defining these methods for this type.

In the following example, we define a type-class named `named_type`. Its contract consists of a single method, `type_name`, which takes an object of the instance type and returns a `string`, representing the name of the type.

The following compiles successfully:

```prolog
class T : named_type where {
    type_name(T) -> string
}.
```

With such a definition in place, concrete types can be assigned as instances of the type class. For example, the type `int64` can be made an instance of `named_type` by implementing `type_name` to simply return the string `"int64"`. For `float64`, the method will return the string `"float64"`, etc.

The following compiles successfully:

```prolog
class T : named_type where {
    type_name(T) -> string
}.

instance int64 : named_type where {
    type_name(_) := "int64"
}.

instance float64 : named_type where {
    type_name(_) := "float64"
}.

assert type_name(3) == "int64".
assert type_name(3.0) == "float64".
```

## Class Declarations

A type-class declaration of the form `class T:C where { Decls }` consists of three elements:

1. The type-class name (`C`).
2. A variable representing an instance of this type (`T`).
3. One or more function declarations (`Decls`), separated by semicolons (`;`), representing the methods.

The instance type must be represented by a free variable, as it abstracts over all possible instances. Placing a concrete type in its place will trigger a compilation error.

```prolog
class int64 : named_type where {
    type_name(int64) -> string
}.
```

```error
Expected instance type to be a free variable in the declaration of class named_type. Found: int64.
```

The declaration of each method must depend on `T`. `T` must appear in at least one of the parameters, and can possibly appear in the return type.

```prolog
class F : foo where {
    method1(F, int64) -> int64;
    method2(string) -> int64
}.
```

```error
Method method2 does not depend on the instance type in the declaration of class foo.
```

## Instance Definitions

An instance of a type-class must implement all methods declared by the type-class, in the same order.

In the following example the instance is missing a method.

```prolog
class F : foo where {
    method1(F, int64) -> int64;
    method2(F, string) -> int64
}.

instance int64 : foo where {
    method1(A, B) := A+B
}.
```

```error
Instance int64 of class foo defines less methods than required by the class.
```

The methods needs to be defined in the same order they were defined in the class.

```prolog
class F : foo where {
    method1(F, int64) -> int64;
    method2(F, string) -> string
}.

instance int64 : foo where {
    method2(_, S) := S;
    method1(A, B) := A+B
}.
```

```error
Expected method1/2 in instance int64 of class foo. Found method2/2.
```

Obviously, it is also illegal to provide too many methods.

```prolog
class F : foo where {
    method1(F, int64) -> int64;
    method2(F, string) -> string
}.

instance int64 : foo where {
    method1(A, B) := A+B;
    method2(_, S) := S;
    method3(N) := N
}.
```

```error
Instance int64 of class foo defines more methods than required by the class.
```

The methods defined in an instance must conform to the type signature they are declared with.

```prolog
class F : foo where {
    method1(F, int64) -> int64;
    method2(F, string) -> string
}.

instance float64 : foo where {
    method1(A, B) := A+B;
    method2(_, S) := S
}.
```

```error
Type mismatch. Expression B::float64 expected to be int64, inferred: float64.
```

## Polymorphic Functions

Type classes allow us to define polymorphic functions. These functions, given a single definition, actually define a "family" of simple functions, which differ only in their argument types.

Consider the `named_type` example given above. Using this type-class we can define a polymorphic function `greet_type` which takes a `string` greeting and an object of any `named_type` and returns a string greeting the type. The following compiles successfully:

```prolog
class T : named_type where {
    type_name(T) -> string
}.

T : named_type =>
declare greet_type(string, T) -> string.

greet_type(Greeting, Obj) := Greeting + ", " + type_name(Obj).

instance int64 : named_type where {
    type_name(_) := "int64"
}.

instance float64 : named_type where {
    type_name(_) := "float64"
}.

assert greet_type("hello", 3) == "hello, int64".
assert greet_type("hola", 3.0) == "hola, float64".
```

A polymorphic function cannot assume anything about the types it receives as parameters, except that they conform to the classes they are declared to conform to. For example, we cannot assume `Obj` is of type `string`.

```prolog
class T : named_type where {
    type_name(T) -> string
}.

T : named_type =>
declare greet_type(string, T) -> string.

greet_type(Greeting, Obj) := Greeting + ", " + Obj.
```

```error
Expression Obj::string expected to be unknown_type1(named_type), inferred: string.
```

When calling a polymorphic function, the compiler checks the assumptions hold for the given types. For example, if we call `greet_type` on a `float64` without first defining `float64` as an instance of `named_type`, we get a compilation error.

```prolog
class T : named_type where {
    type_name(T) -> string
}.

T : named_type =>
declare greet_type(string, T) -> string.

greet_type(Greeting, Obj) := Greeting + ", " + type_name(Obj).

assert greet_type("hola", 3.0) == "hola, float64".
```

```error
Type float64 is not an instance of class named_type, which is an assumption made by greet_type.
```