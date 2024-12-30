- Feature Name: `rust_named_and_default_arguments`
- Start Date: 2024-12-18
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Named and default arguments improve code readability, flexibility and reduce number of errors in function calls. This can be achived when using them by explicitely specifying argument names and merging similar functions into one.

# Motivation
[motivation]: #motivation

Named and default arguments improve both reading and writing of code in several ways:
- explicitly telling what arguments are;
- reduce boilerplate code and need to create a separate function for each case;
- familiar to developers and easy to introduce;
- simplify library API.

Here, it is not clear which argument of insert is position and which is new element itself:
```rust
let mut vec: Vec<usize> = vec![1, 2, 3, 4, 5];
vec.insert(2, 3);
=>
vec.insert(element = 2, at = 3);
```

For instance, these two functions can be merged to one with named & default argument:
```rust
Vec::shrink_to_fit(&mut self)
Vec::shirink_to(&mut self, min_capacity: usize)
    =>
Vec::shrink(&mut self, min_capacity: usize = None)
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Declaring a function with named arguments

For `element`, there will be the same name for both public and in-function use:
```rust
fn insert(idx: usize, pub element: T) {
    ...
}
```
First parameter here is positional and cannot be used as named.

Here, destructured `a` and `b` will be used inside function and `box` will be public one:
```rust
fn insert(idx: usize, pub[box] Box{a, b}: Box) {
    ...
}
```

To call such functions, this syntax can be used:
```rust
insert(1, element: 4);
insert(2, box: new Box(24, 42));
```

While positional arguments should go in exactly the same order as declared in function, named arguments can be specified in an arbitrary order and should go after positional ones. If the arguments in a call go in exact same order as in declaration, we should be allowing to specify named arguments without names to be backward-compatible with older code.

## Declaring a function with default arguments

To declare parameter as optional with default value, it should be made named (`pub`) and the value should be specified after its name or pattern:
```rust
fn insert(index: usize, pub element: usize = 42) {
    ...
}
```

In a call to such function, this argument can be ommited and the default value will be used.

## Positional, named and default arguments

Features explained above can be combined together:
```rust
pub build_house(
    type: Type,
    pub walls: Walls,
    pub[roof] cover: Cover,
    pub interior: Option<Interior> = None
) {
    ...
}
```

```rust
// Valid 
build_house(Type::Commercial, Walls::Concrete, roof: Cover::Aluminum);
build_house(Type::Commercial, walls: Walls::Concrete, interior: Interior::Modern, roof: Cover::Aluminum);

// Invalid
build_house(walls: Walls::Concrete, Type::Commercial, roof: Cover::Wooden);
build_house(Type::Commercial, roof: Cover::Wooden);
```

## Combined with `ref`/`mut` modifiers

```rust
fn insert(idx: usize, pub[box] mut Box{a, b}: Box) {
    ...
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Semantics
Named and default arguments seems to be mostly syntactic sugar. There is probably a case with traits implementation, where we would like to enforce argument names from trait to be consistent.

## API compatability
Backward-compatible changes:
- Converting positional argument to named
- Converting positional/named argument to a default

Backward-incompatible changes:
- Converting named argument to positional
- Changing name of named argument
- Removing argument

## ABI compatability
Named and default arguments should not affect ABI as they should not appear there.

## Type system
Named and default arguments does not change type system. One of the enforcements that should be created though is for default argument value to be compatible with the parameter type.

# Drawbacks
[drawbacks]: #drawbacks

Rust ecosystem is already quite established and such major change will have a big impact on it.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Mandatary named arguments
Languages like Python and Swift require to always specify argument names in function call. This can be disabled though for example in cases when there are only small amount of them.

## Convenient HashMap-like parameter
Languages such as JavaScript, Ruby, Groovy, Clojure support named and default arguments through providing a map into an argument of a function. Map can be defined easily, and even provided inline along in the function call statement. This way they can use named and default arguments in functions. 

```js
function someFunc(arg) {
    alert(arg.foo);
    alert(arg.bar);
}

someFunc({foo: "This", bar: "works!"});
```

## IDE support
With the advancements of code analysis features (such as `rust-analyzer` and IntelliJ Rust), argument names can be shown in function calls automatically, and default arguments placed through auto-completion. This makes this feature less relevant in such cases, reduces code overhead and need to support it in language itself.

## Builder pattern
Builder pattern provides both concise argument names and their defaults if needed. However, it requires a lot of boilerplate for any new argument and strict definition and naming conventions.

```rust
#[derive(Debug, PartialEq)]
pub struct Foo {
    bar: String, // required parameter
    baz: Option<String>, // optional parameter
}

impl Foo {
    pub fn builder() -> FooBuilder {
        FooBuilder::default()
    }
}

#[derive(Default)]
pub struct FooBuilder {
    bar: String,
    baz: Option<String>,
}

impl FooBuilder {
    pub fn new(bar: String) -> FooBuilder {
        // Set the minimally required fields of Foo.
        FooBuilder {
            bar,
            baz: None // default is None
        }
    }

    pub fn bar(mut self, bar: String) -> FooBuilder {
        self.bar = bar;
        self
    }

    pub fn baz(mut self, baz: String) -> FooBuilder {
        self.baz = baz;
        self
    }

    pub fn build(self) -> Foo {
        Foo { bar: self.bar, baz: self.baz }
    }
}
```

## Separate function names
Separate names for each call type is how currently usually libraries handle these cases, when amount of arguments not sufficient enough to introduce a builder. This way we separate cases into multiple functions and also rename them accordingly, as function overloading is not available in Rust.

```rust
Vec::shrink_to_fit(&mut self)
Vec::shirink_to(&mut self, min_capacity: usize)
```

# Prior art
[prior-art]: #prior-art

## Previous proposals

Through the past years there have been several proposals and discussions on this feature.

Proposals:
- 2014-09-23 - Default and positional arguments
    - [Proposal](https://github.com/KokaKiwi/rfcs/blob/default_args/active/0000-default-arguments.md)
    - [Discussion](https://github.com/rust-lang/rfcs/pull/257)
- 2014-09-26 - Wishlist: functions with keyword args, optional args, and/or variable-arity argument (varargs) lists
    - [Discussion](https://github.com/rust-lang/rfcs/issues/323)
- 2015-02-03
    - [Proposal](https://github.com/iopq/rfcs/blob/2e41311ffedacab38f80073deb29044f1ae20f7e/active/0000-keyword-arguments.md)
    - [Discussion](https://github.com/rust-lang/rfcs/pull/805)
- 2015-12-10 - What's the state of named function parameters? 
    - [Discussion](https://www.reddit.com/r/rust/comments/3w7adf/whats_the_state_of_named_function_parameters/)
- 2016-08-08 - [Pre-RFC] named arguments
    - [Proposal & discussion](https://internals.rust-lang.org/t/pre-rfc-named-arguments/3831)
- 2020-07-15 - [Pre-RFC] Named .arguments
    - [Proposal](https://github.com/Aloso/rfcs/blob/named-arguments/text/0000-named-arguments.md)
    - [Discussion](https://internals.rust-lang.org/t/pre-rfc-named-arguments/12730)
- 2022-04-03 - Pre-RFC: Named arguments
    - [Proposal](https://github.com/poliorcetics/named-arguments-rfc/blob/main/0000-named-arguments-rfc.md)
    - [Discussion](https://internals.rust-lang.org/t/pre-rfc-named-arguments/16413)

In other languages:
- Scala - [Named and default arguments for polymorphic object-oriented languages](https://infoscience.epfl.ch/server/api/core/bitstreams/b54cc2d5-9884-4f59-902e-642bad911647/content)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

### What is the actual demand for this feature?
It is not currently clear what is the actual demand for this feature in Rust. The language and compiler have already been stable for the past 10 years, libraries and projects have been successfully evolving without it.

### How standard library should migrate if this feature becomes available?
This is a major change to capabilities of designing libraries API. In some cases, applying principle of named and default arguments can lead to deprecation of functions, major changes to their interface and intended use. This will require gradual update of library, documentation and tutorials.

# Future possibilities
[future-possibilities]: #future-possibilities

## Variadic functions
Although variadic functions do not exist in Rust altogether, named variadic arguments in Swift allow caller to pass arbitrary amount of objects of multiple types.

```swift
mutating func add(_ newContent: Content..., newUsers: User...) {
    content.append(contentsOf: newContent)
    users.append(contentsOf: newUsers)
}
```
