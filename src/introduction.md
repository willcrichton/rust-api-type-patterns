# Rust API Type Patterns

This book is for people who are writing APIs in Rust. In particular, APIs that involve complex abstractions: variadics, state machines, extensible architectures, and so on.

**The central theme is how to enforce API invariants by replacing dynamically-typed abstractions with statically-typed ones.**

## A prototypical example: enums over strings

A basic example of this theme is using enums to represent a finite set of options, rather than a string. Imagine trying to convert a description of a color to an RGB tuple. This API design is error-prone:

```rust
fn color_to_rgb_bad(color: &str) -> Option<(u8, u8, u8)> {
  match color {
    "Red" => Some((255, 0, 0)),
    "Green" => Some((0, 255, 0)),
    "Blue" => Some((0, 0, 255)),
    _ => None
  }
}
```

While this design avoids errors:

```rust
enum PrimaryColor { Red, Green, Blue }
fn color_to_rgb_good(color: PrimaryColor) -> (u8, u8, u8) {
  match color {
    PrimaryColor::Red => (255, 0, 0),
    PrimaryColor::Green => (0, 255, 0),
    PrimaryColor::Blue => (0, 0, 255)
  }
}
```

Specifically, errors could happen either for a API client or the API author:
* **For a API client,** the `color` parameter in the first example is an dynamically-typed (or "stringly-typed") abstraction because the compiler cannot verify whether a call to `color_to_rgb_bad` will always contain one of the three strings. By contrast, in `color_to_rgb_good`, the input is guaranteed to be one of the enum values. Consequently, the `Option` is removed from the type signature because the method can no longer fail.
* **For the API author,** the compiler does not enforce that the implementation of `color_to_rgb_bad` matches on the strings intended by the author. If they wrote `"Rod"` instead of `"Red"`, the error would only arise in a unit test or client bug report. By contrast, the compiler enforces that each enum variants must be one of the three enum values.

## Book structure

The remainder of the book is structured much like the example above. Each chapter follows the pattern:
1. Describe the concepts and constraints of a system in a particular application domain.
2. Show the dynamically-typed and statically-typed implementations of an API for the system.
3. Analyze what errors can happen in each flavor of API.

You can consume this book à la carte, reading individual chapters as they're useful to you. But each chapter does (somewhat) build on the previous ones, so you may enjoy a start-to-finish read as well.
