Monads
======

When programming, we are regularly take a value and perform some operation to get a new value.

Let's say we have the value `5` which is an integer. We could square the value which gives us `25` which is still an
integer, or we could turn the value into a string which would give us `"5"` which is a different type.

```rust
# fn main () {
let twenty_five = 5 * 5;
let five = 5.to_string();

assert_eq!(twenty_five, 25);
assert_eq!(five, "5");
}
```

We might want to perform the same operation on any integer, so we create functions that can take _any_ integer and
return our new value.

```rust
// Don't worry that there are already built in ways to do this, 
// its for explanation only
fn square(input: i32) -> i32 {
    input * input
}

fn to_string(input: i32) -> String {
    format!("{input}")
}
# 
# fn main() {
#     assert_eq!(square(5), 25);
#     assert_eq!(to_string(5), "5".to_string());
# }
```

This shows to types of function:
- A function that maintains the input type: it takes a value of type `T` and returns a value of type `T`.
  We'll refer to this as `f(T) -> T`.
- A function that changes the input type: it takes a value of type `T` and returns a value of type `U`.
  We'll refer to this as `f(T) -> T`.

Typically with functions, we put the value into the function and get a value back out, but what if we reverse this.

What if we have a container for our value, we'll call it `M`. `M` can contain any valid value of type `T` and could be
represented as `M<T>`. We still have our two types of function that can take a value of `T`, but they can't take a value
of `M<T>` because it's a different type.

What we could do instead is give the function to the container and ask it to apply it to the inner type.

If we start with `M<T>` and give it `f(T) -> T` then the result is `M<T>`. But, if we start with `M<T>` and give it 
`f(T) -> U` then the result is `M<U>`.

In practice this might look like this:

```rust
# fn square(input: i32) -> i32 {
#     input * input
# }
# 
# fn to_string(input: i32) -> String {
#     format!("{input}")
# }
# 
struct Container<T> {
    inner_value: T,
}

impl<T> Container<T> {
    fn apply<U, F: FnOnce(T) -> U>(self, f: F) -> Container<U> {
        Container {
            inner_value: f(self.inner_value)
        }
    }
}

# fn main() {
let five = Container { inner_value: 5 };
let result = five.apply(square).apply(to_string);
assert_eq!(result.inner_value, "25".to_string());
# }
```

Why would you do this?
----------------------

Our container, `M`, could contain more types. By delegating the work to the container type, we could allow our container
type to decide how (or even if) the function operates on its internal data.

Let's try creating a container which contains two types, an arbitrary type `T`, and the unit type `()`. In this
container, when applying a function that takes `T` to our container, it only needs to apply it if the container contains
a value of type `T`, otherwis it can ignore it.

```rust
# fn square(input: i32) -> i32 {
#     input * input
# }
# 
# fn to_string(input: i32) -> String {
#     format!("{input}")
# }
# 
# #[derive(Debug, PartialEq)]
enum MaybeContainer<T> {
    ContainsSomething(T),
    ContainsNothing(())
}

impl<T> MaybeContainer<T> {
    fn apply<U, F: FnOnce(T) -> U>(self, f: F) -> MaybeContainer<U> {
        match self {
            Self::ContainsSomething(x) => MaybeContainer::ContainsSomething(f(x)),
            Self::ContainsNothing(()) => MaybeContainer::ContainsNothing(()),
        }
    }
}

# fn main() {
let five = MaybeContainer::ContainsSomething(5);
let result = five.apply(square).apply(to_string);
assert_eq!(result, MaybeContainer::ContainsSomething("25".to_string()));

let nothing: MaybeContainer<i32> = MaybeContainer::ContainsNothing(());
let result = nothing.apply(square).apply(to_string);
assert_eq!(result, MaybeContainer::ContainsNothing(()));
# }
```

This looks a bit familiar right? Let's quickly tidy this up:
- we don't need to write in the unit type as its literally the type representation of nothing
- the `apply` method name isn't very specific, we always need a function that takes `T` and returns `U` (even if they're
  both the same type)
- we'll shorten the name of the container and its varints. It represents the concepts of something or nothing,
  effectivily an optional type

So maybe it should look more like this:


```rust
# fn square(input: i32) -> i32 {
#     input * input
# }
# 
# fn to_string(input: i32) -> String {
#     format!("{input}")
# }
# 
# #[derive(Debug, PartialEq)]
enum Option<T> {
    Some(T),
    None,
}

impl<T> Option<T> {
    fn apply<U, F: FnOnce(T) -> U>(self, f: F) -> Option<U> {
        match self {
            Self::Some(x) => Option::Some(f(x)),
            Self::None => Option::None,
        }
    }
}
# 
# fn main() {
# let five = Option::Some(5);
# let result = five.apply(square).apply(to_string);
# assert_eq!(result, Option::Some("25".to_string()));
# 
# let nothing: Option<i32> = Option::None;
# let result = nothing.apply(square).apply(to_string);
# assert_eq!(result, Option::None);
# }
```

Yeap, we just recreated Rust's `Option` type.

Our groups can contain more than something or nothing. They could contain a type that represents the concept of
successful execution, or an error, eg `Result<T, E>`. In Rust, the Result type generics, `T` and `E` could be the same
concrete type, for example, `String`. In order to differentiate them inside the result we have seperate functions
for applying functions to each; `map()`, and `map_err()`.

Rules / Guidelines
------------------

The important thing for this kind of group is that:
- For a given group, a value representable as a type inside that group, should be convertable to that group without
  its value changing. i.e. there must be a way of going from `T -> m<T>` without the inner value changing.
- There needs to be a way to apply a function to the inner types such that `f(T) -> U` can be applied to `m<T>` to
  produce `m<U>`

Benefits
--------

Encapsulating our values inside groups like this can significantly reduce the cognative overhead of our code, and its
a genuine shame that we've scared people off of this approach with complex naming and annotations.

Here are two piece of code written in typescript that demonstrate the power of nomads. We'll create a function that
divides two numbers, but if the denominator is zero, we'll treat it as no value. First we'll implement the code without
monads:

```typescript
const div = (denominator: number) => (numerator: number): number | null => {
    if (denominator == 0) {
        return null;
    }
    return numerator / denominator;
};

const add = (a: number) => (b: number) => a + b;

const div_zero = div(0);
const div_two = div(2);
const add_one = add(1);

let null_example = div_zero(6);
if (null_example !== null) {
    null_example = add_one(null_example);
}
if (null_example !== null) {
    null_example = div_zero(null_example);
}
if (null_example !== null) {
    // Won't get here
    console.log(null_example);
}

let num_example = div_two(6);
if (num_example !== null) {
    num_example = add_one(num_example);
}
if (num_example !== null) {
    num_example = div_two(num_example);
}
if (num_example !== null) {
    // Prints 2
    console.log(num_example);
}

console.log('done');
```

If we create a proxy for Rust's Option type though (in this case we called it `Maybe`), this code gets much easier to
read.

```typescript
# class Maybe<T> {
#     inner: T | null,
# 
#     public static some(value: T): Maybe<T> {
#         return new Maybe(value);
#     }
#
#     public static none(): Maybe<T> {
#         return new Maybe();
#     }
#
#     private constructor(value: T | null | undefined) {
#         if (value === undefined) {
#             this.inner = null;
#         } else {
#             this.inner = value;
#         }
#     }
#
#     public map<U>(f: (T) -> U): Maybe<U> {
#         if (this.inner == null)  {
#             return new Maybe();
#         }
#         return new Maybe(f(this.inner))
#     }
#
#     public flatten<U>() -> Maybe<U> {
#         if (this.inner instanceof Maybe) {
#             return this.inner.inner === null ?
#                 Maybe.none() :
#                 Maybe.some(this.inner.inner);
#         }
#         return this;
#     }
#
#     public and_then<U>(f: (T) -> Maybe<U>) -> Maybe<U> {
#         return this.map(f).flatten();
#     }
# }
#
const div = (denominator: number) => (numerator: number): Maybe<number> => {
    if (denominator == 0) {
        return Maybe.none();
    }
    return Maybe.some(numerator / denominator)
};

const add = (a: number) => (b: number) => a + b;

const div_zero = div(0);
const div_two = div(2);
const add_one = add(1);

div_zero(6)
    .map(add_one)
    .and_then(div_zero)
    .map(console.log); // Does nothing

div_zero(6)
    .map(add_one)
    .and_then(div_two)
    .map(console.log); // Prints 2

console.log('done');
```

In the first example, we have to deal with two seperate types, which means we constantly need to check what type we're
dealing with. By encapsulating our different types into one group and having the logic that manages what to do for
different types managed inside that group, our code get's far easier to read and comprehend!
