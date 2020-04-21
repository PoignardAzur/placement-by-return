- Feature Name: `variadic_generics`
- Start Date: 2020-04-02
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This RFC proposes to add support for variadic generics by
- allowing arbitrary length generic type parameter and function argument lists that are collected into variadic tuples
- adding the ability to 'spread' a tuple, i.e. turning a tuple into a comma-separated list of it's elements
- adding traits to support proper type checking for variadic tuples
- adding a macro that repeats a given type or expression

# Motivation
[motivation]: #motivation

## Real-World Use Cases

### std::ops::Fn

The [current implementation](https://doc.rust-lang.org/src/core/ops/function.rs.html#69-73) would translate to:

```rust
pub trait Fn<Args@..>: FnMut<Args@..> {
    fn call(&self, param: Args@..) -> Self::Output
    {
        func(param@..)
    }
}
```

### Tuple Helper Functions

A function similar to `std::iter::Iterator::zip` that converts two tuples `(a1, ..., an)`, `(b1, ..., bn)` to `((a1, b1), ..., (an, bn))` could be implemented as:

```rust
fn zip<(A@..), (B@..)>(a: A, b: B)
    -> repeat!{(A, B)}
    where A: SameArityAs<B>
{
    repeat!{(a, b)}
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
#[stable(feature = "rust1", since = "1.0.0")]
impl<Ts@.., T> Hash for (Ts@.., T)
where
    repeat!{T: Hash}@..,
    Last: Hash + ?Sized,
{
    fn hash<S: Hasher>(&self, state: &mut S) {
        let (ref tuple@.., ref last) = *self;
        repeat!{tuple.hash(state)};
        last.hash(state);
    }
}
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

TODO

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Grammer

- [Type and Lifetime Parameters](https://doc.rust-lang.org/stable/reference/items/generics.html): change the type param production to
    - `TypeParam: SimpleTypeParam|VariadicTypeParam|TupleTypeParam`
    - `SimpleTypeParam: OuterAttribute? IDENTIFIER(:TypeParamBounds?)?(=Type)?`
    - `VariadicTypeParam: (SimpleTypeParam)@..|IDENTIFIER@..`
    - `TupleTypeParam: (TupleTypeParamItems?)(:TypeParamBounds?)?(=Type)?`
    - `TupleTypeParamItems: SimpleTypeParam,|SimpleTypeParam(,SimpleTypeParam)+,?|(SimpleTypeParam,)*VariadicTypeParam((,SimpleTypeParam)+)?,?`
- [Where clauses](https://doc.rust-lang.org/stable/reference/items/generics.html#where-clauses):
    - TODO
- [Type expressions](https://doc.rust-lang.org/stable/reference/types.html#type-expressions): add
    - `TupleSpreadType: Type@..` where the given Type must resolve to a tuple type
- [Expressions](https://doc.rust-lang.org/stable/reference/expressions.html): add
    - `TupleSpreadExpression: Expression@..` where the given expression must have a tuple type
- [Tuple patterns](https://doc.rust-lang.org/stable/reference/patterns.html#tuple-patterns): change the tuple pattern grammer to
    - `TuplePatternItems: Pattern,|Pattern(,Pattern)+,?|(Pattern,)* VariadicTuplePatternItem ((, Pattern)+ ,? )?`
    - `VariadicTuplePatternItem: ..|IDENTIFIER@..|TuplePattern@..|(Pattern)@..`

## Library

Add the following trait to the standard library:
```rust
trait SameArityAs<(T@..)> {}
impl SameArityAs<()> for () {}
impl<(A, As@..), (B, Bs@..)> SameArityAs<(A, As@..)> for (B, Bs@..)
    where As: SameArityAs<Bs>
{}
```

Add the ```repeat!``` macro that, for every element in one or more tuples, repeats an expression or a type into a tuple expression or tuple type. The macro allows for a short-hand syntax that expands all mentioned identifiers of tuple type and another syntax that allows to explicitly specify all tuples that should be expanded. Given tuples A and B:
```rust
repeat!{(A, B)}                    // => ((A1, B2), (A2, B2))
repeat!(for a, b in A, B {(a, b)}) // => ((A1, B2), (A2, B2))
repeat!(for a in A {(a, B)})       // => ((A1, B), (A2, B))
```
Here, the first two examples would require A and B to be constrained using the above trait.

## Type Checking

TODO

Basic idea: each variadic tuple type, expression or pattern has arity >= N. A variadic tuple pattern of arity >= N may only match a variadic tuple of arity >= M if M >= N.

```rust
fn zip_and_do_sth<(A@..), (B@..)>(a: A, b: B)
{
    do_sth(repeat!{(a, b)}@..) // => ERROR: a and b may have different lengths
}

fn zip_and_do_sth<(A@..), (B@..)>(a: A, b: B)
    where A: SameArityAs<B>
{
    do_sth(repeat!{(a, b)}@..) // OK
}
```

```rust
fn foo<(A@..)>((a, as@..): A)              // => ERROR: pattern matches 1 or more elements, but type may contain 0 elements
fn foo<(A, As@..)>((a, as@..): (A, As@..)) // => OK
fn foo<(A@..)>((a, b): A)                  // => ERROR: trying to match variadic tuple with non-variadic tuple pattern
```

## Examples

### Paranthesis

The following examples show how paranthesis affect tuple spreads:
```rust
(1, t@.., 1)        // => (1, t1, t2, [...], tn, 1)
(1, (t)@.., 1)      // => (1, t1, t2, [...], tn, 1)
(1, (t,)@.., 1)     // => (1, t, 1)
(1, (t@..), 1)      // => (1, (t1, t2, [...], tn), 1)
```

### Generic Type Parameters and Function Signatures

The following examples show how generic type parameter lists and function signatures are affected by different definitions. This is also relevant for how a function pointer to a specific variadic generic is taken.
NOTE: in the following examples the identifiers in the expanded example are internal compiler representation i.e. not available for use in the original code
```rust
// single type parameter pack
fn foo<A@..>(a: A@..)       // => fn foo<A1, A2>(a1: A1, a2: A2), a = (a1, a2)
fn foo<A@..>(a: A)          // => fn foo<A1, A2>(a: (A1, A2))
fn foo<(A@..)>(a: A)        // => fn foo<(A1, A2)>(a: (A1, A2))
fn foo<A>((a@..): A)        // => Error: A must be a variadic tuple type

// multiple type parameter packs
fn foo<A@.., B@..>(a: A@.., b: B@..)               // => ERROR: multiple variadic generic type parameters
fn foo<(A@..), B@..>(a: A@.., b: B@..)             // => ERROR: multiple variadic function arguments
fn foo<(A@..), B@..>(a: A, b: B@..)                // => fn foo<(A1, A2), B1, B2>(a: (A1, A2), b1: B1, b2: B2), b = (b1, b2)
fn foo<(A@..), (B@..)>((a, b): repeat!{(A, B)}@..) // => ERROR: A and B are not guaranteed to have the same arity
fn foo<(A@..), (B@..)>((a, b): repeat!{(A, B)}@..) // => fn foo<(A1, A2), (B1, B2)>((a1, b1): (A1, B1), (a2, b2): (A2, B2)) where ...
    where A: SameArityAs<B>

// return types
fn foo<A@..>(a: A) -> A@..               // => ERROR: a function return type cannot be a plain tuple spread
fn foo<A@..>(a: A) -> A                  // => fn foo<A1, A2>(a: (A1, A2)) -> (A1, A2)
fn foo<A>(a: A) -> (A@..)                // => ERROR: A may not be a tuple type
fn foo<A@..>(a: A) -> Option<A@..>       // => ERROR: Option is not a variadic generic
fn foo<A@..>(a: A) -> repeat!{Option<A>} // => fn foo<A1, A2>(a: (A1, A2)) -> (Option<A1>, Option<A2>)
fn foo<(A@..), (B@..)>(a: A, b: B) -> repeat!{Result<A, B>} // => ERROR: A and B may have different arities
fn foo<(A@..), (B@..)>(a: A, b: B) -> repeat!{Result<A, B>} // => fn foo<A1, A2, B1, B2>(a: (A1, A2), b: (B1, B2)) -> (Result<A1, B1>, Result<A2, B2>) where ...
    where A: SameArityAs<B>

// trait bounds
fn foo<A@..: Bound>(a: A@..)   // => ERROR: a bound on a tuple spread is not allowed
fn foo<(A: Bound)@..>(a: A@..) // => fn foo<A1: Bound, A2: Bound>(a: (A1, A2))
fn foo<(A@..): Bound>(a: A@..) // => fn foo<(A1, A2): Bound>(a: (A1, A2))

// default type parameters
fn foo<A@.. = Bar>(a: A@..)   // => ERROR: parameter packs cannot be defaulted
fn foo<(A = Bar)@..>(a: A@..) // => fn foo<A1 = Bar, A2 = Bar>(a: (A1, A2))
fn foo<(A@..) = Bar>(a: A@..) // => fn foo<(A1, A2) = Bar>(a: (A1, A2))
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

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities

## Recursion for variadic generics

Allow for recursive calls in variadic generics. This could be made possible function specialization:
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

## Fold expressions

TODO

## Traits using const generics

E.g. a trait to check for a specific arity:

```rust
trait HasArity<const N: usize> {}
impl HasArity<0> for () {}
impl<A, As@..> HasArity<N> for (A, As@..)
    where As: HasArity<N-1>
{}
```
