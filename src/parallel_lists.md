# Parallel Lists

Another common pattern in system design is to have two lists of pairwise-related (or "parallel") elements. For example, consider `println`:

```rust,ignore
println!("#{:03} {}: {}", id, user, message);
```

The first list is the format directives (e.g. `{}`) in the format string. The second list is the inputs (e.g. `id`) to `println!`. These lists must have the same length.

As another example, consider routing in a web server:

```rust,ignore
#[route("/user/:id/messages/:message_id")]
fn user_message(id: Id, message_id: MessageId) {
 /* .. /
}
```

The first list is the routing variables (e.g. `:id`) and the second is the function parameters (e.g. `id: Id`). These lists must have the same length and corresponding types.

I'll show you how to use **heterogeneous lists** (H-lists), or lists of values of different types, to implement APIs with parallel lists. Specifically, we will implement a type-safe printf. For a real-world example, you can check out how the [warp framework](https://docs.rs/warp/) uses H-lists to do type-safe web routing.


## Core mechanism: HList

First, we need to understand H-lists. An H-list is a sequence of values of potentially different types. For example, `[1, "a", true]` is an H-list, but not a valid Rust vector. H-lists are implemented in Rust using a linked-list style:

```rust,ignore
struct HNil;
struct HCons<Head, Tail> {
  head: Head,
  tail: Tail
}

let example: HCons<i32, HCons<bool, HNil>> =
  HCons{head: 1, tail: HCons{head: true, tail: HNil}};
```

The key idea is that the type of an H-list changes every time you make a change to it. By contrast, if you push to a `Vec<T>`, the type of the vector stays the same.

Just like Rust has `vec![]`, we can use the [frunk](https://github.com/lloydmeta/frunk#hlist) crate to get an `hlist!` macro.

```rust,ignore
let example = hlist![1, true]; // same as above
```

## Setting up printf

Let's go back to the ingredients of printf. We need a format string and an argument list. The key idea is to represent both with an H-list, and carefully use Rust's traits to ensure our desired property: the number of arguments should match the number of holes.

First, to represent format strings, we will have a sequence of structs that represent each part of the string.

```rust,ignore
pub struct FString(&'static str);
pub struct FVar;

// Assume that we compile "Hello {}! The first prime is {}" into this code.
// That would be a simple syntactic transformation.
let example = hlist![
  FString("Hello "), FVar, FString("! The first prime is "), FVar
];
```

To represent arguments, we will use a matching H-list of values. For example:

```rust,ignore
let args = hlist!["world", 2];
```

Then, our goal is to create a function `format` such that this is true:

```rust,ignore
assert_eq!(
  example.format(args),
  "Hello world! The first prime is 2"
);
```

And this should be a compile-time (NOT run-time) error:

```rust,ignore
example.format(hlist!["Only one arg"]);
```

## The Format trait

First, we need to define the signature of our `format` function.

```rust,ignore
trait Format<ArgList> {
  fn format(&self, args: ArgList) -> String;
}
```

Here, `self` is the H-list of the format directives, and `ArgList` is the H-list of the variadic arguments. `Format` need to take `ArgList` as a type parameter, because its type will change as we remove elements from the `ArgList` list.

Now, we proceed to implement the `Format` trait by cases. First, the base case for reaching the end of the format list `HNil`:

```rust,ignore
impl Format<HNil> for HNil {
  fn format(&self, _args: HNil) -> String {
    "".to_string()
  }
}
```

This impl says that when we reach the end of a format list, just return the empty string. And the only argument we will accept is an empty argument list. Combined with the next impls, this inductively ensures that extra arguments are not accepted.

Next, we will implement `FString`. This implementation should use the string constant contained in the `FString` struct, and combine it recursively with the rest of the format list. We don't use variadic arguments for `FString`, so they get passed along. In Rust, this English specification becomes:

```rust,ignore
impl<ArgList, FmtList> Format<ArgList>
for HCons<FString, FmtList>
where FmtList: Format<ArgList>
{
  fn format(&self, args: ArgList) -> String {
    self.head.0.to_owned() + &self.tail.format(args)
  }
}
```

Note that we have to add `FmtList: Format<ArgList>` to ensure the recursive call to `self.tail.format` works. Also note that we aren't implementing `Format` directly on `FString`, but rather on an H-list containing `FString`.

Finally, the most complex case, `FVar`. We want this impl to take an argument from the `ArgList`, then format the remaining format list with the remaining arguments.

```rust,ignore
impl<T, ArgList, FmtList> Format<HCons<T, ArgList>>
for HCons<FVar, FmtList>
where
  FmtList: Format<ArgList>,
  T: ToString,
{
  fn format(&self, args: HCons<T, ArgList>) -> String {
    args.head.to_string() + &self.tail.format(args.tail)
  }
}
```

Be careful to observe which H-list is being accessed by `head` and `tail`. Here, the `args` H-list provides the data to fill the hole via `args.head`.

## Checking our properties

With this implementation, our correct example successfully compiles and runs:

```rust,ignore
let example = hlist![
  FString("Hello "), FVar, FString("! The first prime is "), FVar
];
assert_eq!(
  example.format(hlist!["world", 2]),
  "Hello world! The first prime is 2"
);
```

What about our incorrect example? If we write this:

```rust,ignore
example.format(hlist!["just one arg"]);
```

This code fails to compile with the error:

```ignore
error[E0308]: mismatched types
  --> src/printf.rs:48:18
   |
48 |   example.format(hlist!["just one arg"]);
   |                  ^^^^^^^^^^^^^^^^^^^^^^
   |                  expected struct `Cons`, found struct `HNil`
   |
   = note: expected struct `HCons<_, HNil>`
              found struct `HNil`
```

While the error is enigmatic, our mistake is at least correctly caught at compile-time. This is because Rust deduces that `example.format()` expects an H-list of the shape `HCons<_, HCons<_, HNil>>`, but it finds `HNil` too soon in our 1-element H-list. A similar error occurs when providing too many args.

Stupendous! We have successfully implemented a type-safe printf using H-lists and traits.

## Extending our abstraction

Right now, our `Format` function just checks that the format list and argument list are the same length. We could extend our format structures, for example to ensure that an `FVar` must be a particular type, or must use `Debug` vs. `Display`. Here's the sketch of such a strategy:

```rust,ignore
use std::marker::PhantomData;

// Add flags for whether using Display or Debug
pub struct FDisplay;
pub struct FDebug;

// Use a type parameter with PhantomData to represent the intended type
pub struct FVar<T, Flag>(PhantomData<(T, Flag)>);

// Now, T has to be the same between the format list and arg list
// Also, FDisplay flag requires that `T: Display`
impl<T, ArgList, FmtList> Format<HCons<T, ArgList>>
for HCons<FVar<T, FDisplay>, FmtList>
where
  FmtList: Format<ArgList>,
  T: Display,
{
  fn format(&self, args: HCons<T, ArgList>) -> String {
    // using format! is cheating, but you get the idea
    format!("{}", args) + &self.tail.format(args.tail)
  }
}

// Similar impl for `T: Debug` when `FDebug` is used
```

With this approach, if our format list and arg list differ in type:

```rust,ignore
let fmt = hlist![FString("n: "), FVar::<i32, FDisplay>(PhantomData)];
fmt.format(hlist!["not a number"]);
```

Then the code will not compile with the error, `&'static str is not i32`.
