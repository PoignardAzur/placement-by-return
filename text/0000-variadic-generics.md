- Feature Name: `variadic_generics`
- Start Date: 2020-04-02
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This RFC proposes to add support for variadic generics as an extension of Rust's pattern matching syntax:
- allow `identifier@..` in more places like generic type parameter lists and function arguments
- allow for a tuple pattern like syntax in generic type parameter lists
- add the ability to give RangeFull expressions in tuple patterns an identifier

# Motivation
[motivation]: #motivation

## Real-World Use Cases

### std::ops::Fn

The [current implementation](https://doc.rust-lang.org/src/core/ops/function.rs.html#69-73) would translate to:

```rust
pub trait Fn<Args@..>: FnMut<Args@..> {
    fn call(&self, (param: Args)@..) -> Self::Output
    {
        func(param@..)
    }
}
```

### Tuple Helper Functions

A function similar to `std::iter::Iterator::zip` that converts two tuples `(a1, ..., an)`, `(b1, ..., bn)` to `((a1, b1), ..., (an, bn))` could be implemented as:

```rust
fn zip<(A@..), (B@..)>((a@..): (A@..), (b@..): (B@..))
    -> ((A, B)@..)
    where (A@..): SameArityAs<(B@..)>
{
    ((a, b)@..)
}
```

### Tuple Hash Implementation

The current implementation of the `Hash` trait requires a macro which is used to generate implementations for different tuple arities. This
- limits the implementation to be generated for a finite number of types,
- increases the size of the generated code and
- makes the documentation unreadable.
The [current implementation](https://github.com/rust-lang/rust/blob/3982d3514efbb65b3efac6bb006b3fa496d16663/src/libcore/hash/mod.rs#L625) looks like this:

```rust
macro_rules! impl_hash_tuple {
    () => (
        #[stable(feature = "rust1", since = "1.0.0")]
        impl Hash for () {
            fn hash<H: Hasher>(&self, _state: &mut H) {}
        }
    );

    ( $($name:ident)+) => (
        #[stable(feature = "rust1", since = "1.0.0")]
        impl<$($name: Hash),+> Hash for ($($name,)+) where last_type!($($name,)+): ?Sized {
            #[allow(non_snake_case)]
            fn hash<S: Hasher>(&self, state: &mut S) {
                let ($(ref $name,)+) = *self;
                $($name.hash(state);)+
            }
        }
    );
}

macro_rules! last_type {
    ($a:ident,) => { $a };
    ($a:ident, $($rest_a:ident,)+) => { last_type!($($rest_a,)+) };
}
```

This implementation will translate to:

```rust
impl<Ts@.., T> Hash for (Ts@.., T)
where
    (T: Hash)@..,
    Last: Hash + ?Sized,
{
    fn hash<S: Hasher>(&self, state: &mut S) {
        let (ref tuple@.., ref last) = *self;
        ((tuple.hash(state))@..);
        last.hash(state);
    }
}
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

TODO

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Nomenclature

- For generic type parameter `IDENTIFIER@..` or `(IDNTIFIER@..)` the identifier declares a _type parameter pack_
- A type expression `<Type>@..` is called a _type pack expansion_, in this context `..` is called _expansion operator_.
- A variable of parameter pack type is called a _variable parameter pack_
- The expression `<Expression>@..` is called a _variable pack expansion_, in this context `..` is called _expansion operator_.
- In `fn foo<A@..>((a: A)@..)` the syntax `(a: A)@..` is called an _function argument pack expansion_.
- When making a statement about a _prameter pack_ it may either be a type or a variable parameter pack.
- A parameter pack is _within the scope_ of an expansion operator if it's mentioned within the type/expression of the pack expansion and it is not _within the scope_ of any inner pack expansion.
- A parameter pack not within the scope of an expansion operator is _free_.

## Grammer

- [Type and Lifetime Parameters](https://doc.rust-lang.org/stable/reference/items/generics.html): change the type param production to
    - `TypeParam: SimpleTypeParam|VariadicTypeParam|TupleTypeParam`
    - `SimpleTypeParam: OuterAttribute? IDENTIFIER(:TypeParamBounds?)?(=Type)?`
    - `VariadicTypeParam: (SimpleTypeParam)@..|IDENTIFIER@..`
    - `TupleTypeParam: (TupleTypeParamItems?)(:TypeParamBounds?)?(=Type)?`
    - `TupleTypeParamItems: SimpleTypeParam,|SimpleTypeParam(,SimpleTypeParam)+,?|(SimpleTypeParam,)*(SimpleTypeParam)@..((,SimpleTypeParam)+)?,?`
- [Type expressions](https://doc.rust-lang.org/stable/reference/types.html#type-expressions): add `TypePackExpansion: IDENTIFIER@..|TupleType@..|(Type)@..` where the given Type must mention at least one parameter pack
- [Tuple patterns](https://doc.rust-lang.org/stable/reference/patterns.html#tuple-patterns): change the tuple pattern grammer to
    - `TuplePatternItems: Pattern,|Pattern(,Pattern)+,?|(Pattern,)* VariadicTuplePatternItem ((, Pattern)+ ,? )?`
    - `VariadicTuplePatternItem: ..|IDENTIFIER@..|TuplePattern@..|(Pattern)@..`
- [Functions](https://doc.rust-lang.org/stable/reference/items/functions.html?highlight=function#functions) (using the grammer from [this post](https://internals.rust-lang.org/t/pre-rfc-variadic-generics/12123/21?u=alexanderlinne)):
    - `FnArgs: FnArg(,FnArg)*,?|(FnArg,)*VariadicFnArg((,FnArg)+)?,?`
    - `FnArg: SelfValue|SelfRef|Regular`
    - `Regular: FnSigInput`
    - `FnSigInput: (Pattern:)?Type`
    - `VariadicFnArg: Type@..|(FnSigInput)@..`

## Traits

Add the following trait to the standard library:
```rust
trait SameArityAs<(T@..)> {}
impl SameArityAs<()> for () {}
impl<(A, As@..), (B, Bs@..)> SameArityAs<(A, As@..)> for (B, Bs@..)
    where (As@..): SameArityAs<(Bs@..)>
{}

trait HasArity<const N: usize> {}
impl HasArity<0> for () {}
impl<A, As@..> HasArity<N> for (A, As@..)
    where (As@..): HasArity<N-1>
{}
```

## Examples

### Paranthesis

The following examples show how paranthesis affect unfoldings:
```rust
(1, t@.., 1)        // => (1, t1, t2, [...], tn, 1)
(1, (t)@.., 1)      // => (1, t1, t2, [...], tn, 1)
(1, (t@..), 1)      // => (1, (t1, t2, [...], tn), 1)
(1, (t,)@.., 1)     // => (1, (t1,), (t2,), [...], (tn,), 1)
```

### Generic Type Parameters and Function Signatures

The following examples show how generic type parameter lists and function signatures are affected by different definitions. This is also relevant for how a function pointer to a specific variadic generic is taken.
NOTE: in the following examples the identifiers in the unfolded example are internal compiler representation i.e. not available for use in the original code
```rust
// single type parameter pack
fn foo<A@..>((a: A)@..)     // => fn foo<A1, A2>(a1: A1, a2: A2)
fn foo<A@..>(a: (A@..))     // => fn foo<A1, A2>(a: (A1, A2))
fn foo<(A@..)>(a: (A@..))   // => fn foo<(A1, A2)>(a: (A1, A2))
fn foo<A>((a@..): A)        // => Error: A must be a type pack unfolding

// multiple type parameter packs
fn foo<A@.., B@..>((a: A)@.., (b: B)@..)         // => ERROR: multiple type parameter packs
fn foo<(A@..), B@..>((a: A)@.., (b: B)@..)       // => ERROR: multiple function argument expansions
fn foo<(A@..), B@..>(a: (A@..), (b: B)@..)       // => fn foo<(A1, A2), B1, B2>(a: (A1, A2), b1: B1, b2: B2)
fn foo<(A@..), (B@..)>(((a, b): (A, B))@..)      // => ERROR: A and B may have different arities
fn foo<(A@..), (B@..)>(((a, b): (A, B))@..)      // => fn foo<(A1, A2), (B1, B2)>((a1, b1): (A1, B1), (a2, b2): (A2, B2)) where ...
    where (A@..): SameArityAs<(B@..)>
fn foo<(A@..), (B@..)>(((a, b)@..): ((A, B)@..)) // => ERROR: A and B may have different arities
fn foo<(A@..), (B@..)>(((a, b)@..): ((A, B)@..)) // => fn foo<(A1, A2), (B1, B2)>(((a1, b1), (a2, b2)): ((A1, B1), (A1, B2))) where ...
    where (A@..): SameArityAs<(B@..)>

// return types
fn foo<A@..>(a: (A@..)) -> A@..             // => ERROR: a function can not return a parameter pack
fn foo<A@..>(a: (A@..)) -> (A@..)           // => fn foo<A1, A2>(a: (A1, A2)) -> (A1, A2)
fn foo<A>(a: A) -> (A@..)                   // => ERROR: A is not a type parameter pack
fn foo<A@..>(a: (A@..)) -> Option<A@..>     // => ERROR: Option is not a variadic generic
fn foo<A@..>(a: (A@..)) -> (Option<A>@..)   // => fn foo<A1, A2>(a: (A1, A2)) -> (Option<A1>, Option<A2>)
fn foo<(A@..), (B@..)>(a: (A@..), b: (B@..)) -> (Result<A, B>@..) // => ERROR: A and B may have different arities
fn foo<(A@..), (B@..)>(a: (A@..), b: (B@..)) -> (Result<A, B>@..) // => fn foo<A1, A2, B1, B2>(a: (A1, A2), b: (B1, B2)) -> (Result<A1, B1>, Result<A2, B2>) where ...
    where (A@..): SameArityAs<(B@..)>

### Function Implementations

```rust
fn zip_and_do_sth<(A@..), (B@..)>((a: A)@.., (b: B)@..)
{
    do_sth((a, b)@..) // => ERROR: a and b may have different lengths
}

fn zip_and_do_sth<(A@..), (B@..)>((a: A)@.., (b: B)@..)
    where (A@..): SameArityAs<(B@..)>
{
    do_sth((a, b)@..) // OK
}
```

# Drawbacks
[drawbacks]: #drawbacks

## Limitations

### Recursive function calls

Due to the lack of function specialization in Rust, recursive function calls in variadic generics must be forbidden. Two possible solutions are discussed [here](#recursion-for-variadic-generics). How recursion can be implemented using a trait is shown in [rust-lang/rfcs#2775](https://github.com/rust-lang/rfcs/issues/2775) in [this example](https://github.com/fredpointzero/rfcs/blob/variadic_tuples/text/0000-variadic-tuples.md#recursion).

## Alternatives

This section discusses alternative designs and design choices. All of those are designs proposed in earlier RFCs. Summaries of those RFCs can be found [here](#prior-rfcs).

### Cons Lists

While this design allows for variadic generics with little to no changes to Rust's syntax, the ease of use when writing a function and ease of comprehension when reading code is not great. ([example](https://github.com/Woyten/rfcs/blob/master/text/0000-generalized-arity-tuples.md#guide-level-explanation))

### Expansion of tuples into a list of types

Idea: `(..(A, B))` would be similar to `(A, B)`.

Advantage: this would not require the special case of parameter packs.

Disadvantage: the expansion of `(A, B)@..` to `(A1, B1), (A2, B2), [...]` would not be possible.

### Use `(..(A, B))` to declare tuples of the same arity

Idea: `(..(A, B))` would declare type parameter packs `A` and `B` with equal arity. If two parameter packs are used within one expansion, they must have been declared together.

Disadvantages: Much of the criticism from the prior section would also apply to this idea. Additionally, pack expansions with multiple parameter packs already require them to have the same arity and this design would make it impossible to e.g. use a parameter pack declared on a struct impl and a parameter pack declared on a function within that impl within the same expansion.

Advantages: when calling such a generic function the compiler would issue an error referencing the function header instead of the implementation. This could be solved by providing a const function that returns the arity of a parameter pack and allowing boolean expressions in `where` clauses (see [the Future Possibilities section](#future-possibilities)):
```rust
fn zip<(A@..), (B@..)>([...]) -> [...]
    where arity<A@..>() == arity<B@..>()
```

### `..T` instead of `T@..` for Types

Instead of `fn foo<T@..>((a: T)@..)` the syntax could be `fn foo<..T>(a: ..T)`. This was already proposed in prior RFCs and was generally accepted as a possible design choice in discussions.

Advantages `..T`:
- `@ ..` is overloaded with a new meaning. Currently in Rust, `@ ..` is always used to bind an identifier to a match in a pattern. The present RFC proposes to also allow `@ ..` in expression where it means that the expression on the left is expanded at the position of `..`.
- Is is less noisy and looks generally cleaner.

Advantages of `@ ..`:
- Using `@ ..` is consistent with how parameter packs are matches in tuples and the whole proposed syntax is coherent with range expressions and patterns.
- `..T` may be confused with range expressions while `@ ..` is already used in a similar way.
- It allows to e.g. use matches in generic type parameter lists: given `<T@(Ts@..)>`, `T` matches the whole tuple type while `Ts` matches the type parameter pack. Otherwise this would look like this: `T@(..Ts)`.

### Use `t..` or `..t` for pack expansions

Instead of `func(param @ ..)` use `func(param..)`. This looks exactly like a range expression while it has a very different meaning.

### `let (tup @ ..) = [...]` declares a tuple of references

Advantage: could be considered more consistent with slice patterns

Disadvantages:
- A special syntax to create parameter packs would be needed.
- It is not intuitive why tup should be `(&T1, &T2, [...])` instead of `&(T1, T2, [...])`.
- A parameter pack is more flexible. Both `(&T1, &T2, [...])` and `&(T1, T2, [...])` can easily be created from the matched parameter pack.

### Use `for [...] @in [...]` for pack expansions

Disadvantages:
- Easy to confuse with normal for loops which (in general) do not generate code.
- In each iteration the variable may have a different type which is unintuitive compared to normal for loops.

# Prior art
[prior-art]: #prior-art

## C++ Variadic Templates and Fold Expressions

### Definition

From [cppreference (parameter pack)](https://en.cppreference.com/w/cpp/language/parameter_pack):
> A template parameter pack is a template parameter that accepts zero or more template arguments (non-types, types, or templates). A function parameter pack is a function parameter that accepts zero or more function arguments.
>
> A template with at least one parameter pack is called a variadic template.

From [cppreference (fold expressions)](https://en.cppreference.com/w/cpp/language/fold):
> [A fold expression r]educes (folds) a parameter pack over a binary operator. 

### Examples

NOTE: All of the folowing examples are taken from [cppreference](https://en.cppreference.com/w/cpp/language/parameter_pack); the comments are added.

```cpp
template<class ... Types> struct Tuple {};
//       ^^^^^^^^^^^^^^^ defines a template parameter pack

template<class ... Types> void f(Types ... args);
//                               ^^^^^^^^^^^^^^ defines a function parameter pack
```

- A class template parameter pack may only be at the last position in the list of template arguments.
- A function template parameter pack may be at any position given all following parameters are defaulted or can be deduced from function parameters.

```cpp
template<class ...Us> void f(Us... pargs) {}
template<class ...Ts> void g(Ts... args) {
    f(&args...);
//    ^^^^^^^^ is a pack expansion with pattern "&args".
}
```

- If two parameter packs appear in a pattern, they are expanded simultaneously and must have the same length.
- For nested pack expansions, the innermost is expanded first. Each expansions must mention at least one parameter pack that is not within the scope of an inner expansion.
- The [cppreference page](https://en.cppreference.com/w/cpp/language/parameter_pack) has a full list of all locations where pack expansions may appear.

```cpp
template<typename... Args>
bool all(Args... args) { return (... && args); }
//                               ^^^^^^^^^^^ is a unary fold

template<typename ...Args>
int sum(Args&&... args) {
    return (args + ... + (1 * 2));
//          ^^^^^^^^^^^^^^^^^^^^ is a binary fold    
}
```

- A unary fold may only be used with a parameter pack of lenth zero is the operator is `&&`, `||` or `,`.
- `(... && args)` and `(args && ...)` have inverted bracketing.

## Prior RFCs

In this section some previous attempts at implementing variadic generics in Rust are summarized. All further RFCs either proposed the same as one of the following RFCs or a mixture of them.

### Variadic Generics ([rust-lang/rfcs#376](https://github.com/rust-lang/rfcs/issues/376), [rust-lang/rust#10124](https://github.com/rust-lang/rust/issues/10124))

- When `..T` is declared, `T` is a tuple type storing the types given for `..T`. This avoids C++'s special case of parameter packs.
- A tuple type preceded by `..` is expanded to a comma-separated list of types i.e. `(..(A, B, C)) => (A, B, C)`.
- The same syntax is used for tuples of values i.e. `(..(true, false)) => (true, false)`.
- A variable capturing a variable number of argument is declared as `..x: T` where `T` is a tuple type. Destructuring of a tuple is written as `let (head, ..tail) = foo`.
- The syntax `impl<..T: Trait>` is considered for type bounds where this could either mean that the tuple type `T` or that each type in `T` have to satisfy `Trait`.

Advantages and drawbacks: see [here](#expansion-of-tuples-into-a-list-of-types)

### Add Generalized Arity Tuples ([rust-lang/rfcs#2702](https://github.com/rust-lang/rfcs/issues/2702), [Rendered](https://github.com/Woyten/rfcs/blob/master/text/0000-generalized-arity-tuples.md))

- Use recursive types to represent variadic tuples. `(T1, (T2, T3))` is identified with `Tuple<T1, Tuple<T2, Tuple<T3, ()>>>`.

Drawbacks: see [here](#cons-lists)

### Variadic Tuples ([rust-lang/rfcs#2775](https://github.com/rust-lang/rfcs/issues/2775), [Rendered](https://github.com/fredpointzero/rfcs/blob/variadic_tuples/text/0000-variadic-tuples.md), [Internal's thread](https://internals.rust-lang.org/t/pre-rfc-variadic-tuple/10878/68))

- `(..T)` declares a variadic tuple type, `(..(T1, T2, ..., Tn))` declares n variadic tuple types that must have the same arity, both may be used as generic type parameters
- a type expression `Expr(T)` containing a variadic tuple type is expanded by `..Expr(T)` (the RFC talks about expressions, but given the examples I think only type expression are meant.)
- `let (tup @ ..) = (1, 2, 3, 4)` declares a variadic tuple `tup` of references
- `for (ref key, map) <KEY, VALUE> @in (k, maps) <K, V>` ([full example](https://github.com/fredpointzero/rfcs/blob/variadic_tuples/text/0000-variadic-tuples.md#iterating-over-variadic-tuple)) iterates over a variadic tuple. Multiple variadic tuples must have the same arity i.e. they must have been declared together.
- explicitly disallows recursion within variadic generic functions

Advantages and drawbacks: see [here](#alternatives)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities

## Parameter pack parameter packs

A syntax similar to
```rust
fn join<(T@..)@..>(((t@..): T)@..) -> ((T@..)@..)
{
    ((t@..)@..)
}
```
which would allow for an arbitrary number of tuples of arbitrary size. It would need examination as to wether this would have actual real-world use cases and wether any readable and unambiguous syntax could be found.

## Recursion for variadic generics

Allow for recursive calls in variadic generics. This could be made possible using either function specialization or `if const`:
- With function specialization the implementation would look like this:
```rust
const fn arity<T, Ts@..>() -> usize
{
    1 + arity<Ts@..>()
}

const fn arity<>() -> usize
{
    0
}
```
- With `if const` the above function would look like this:
```rust
const fn arity<T, Ts@..>() -> usize
{
    if const intrinsics::type_id::<(Ts@..)>() == intrinsics::type_id::<(,)>() {
        0
    } else {
        1 + arity<Ts@..>()
    }
}
```

## Fold expressions

TODO
