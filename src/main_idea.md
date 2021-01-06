# The Main Idea

An API is a computational representation of real-world concepts, and **a good API has a small distance between the representation and the concept**. This begs the question: what defines such a distance?

## Principles of representation

Consider the example from the last chapter. We want to define a data type that represents the three primary colors. We could do this as either a string or an enum:

```rust,ignore
fn color_to_rgb_bad(color: &str) // ...

enum PrimaryColor { Red, Green, Blue }
fn color_to_rgb_good(color: PrimaryColor) // ..
```

Here, an enum is preferable to a string because **there is a one-to-one map from elements of the representation to elements of the concept**. By contrast, the type `&str` includes many more strings than primary colors. Because `&str` is not one-to-one, the `color_to_rgb_bad` function must handle the additional case of elements in the representation that don't exist in the concept.

Previously, we justified the enum design in terms of potential errors avoided. But the point is that these errors are symptoms of a deeper issue: cardinality mismatch. And when we explicitly articulate this principle, then we can more easily apply it to new situations.

For example, cardinality mismatch can happen in multiple ways: consider a situation where an API's representation might contain _fewer_ elements than those in the concept. Consider a variant of Rust's [`fs::read_to_string`](https://doc.rust-lang.org/std/fs/fn.read_to_string.html) function which returns a `bool` instead of `Result<String, io::Error>`.

```rust
use std::{io::Read, path::PathBuf, fs::File};
fn read_to_string(path: String, buffer: &mut String) -> bool {
  let mut file = if let Ok(f) = File::open(PathBuf::from(path)) {
    f
  } else {
    return false;
  };

  if !file.read_to_string(buffer).is_ok() {
    return false;
  }

  return true;
}
```

In this API design, the `bool` represents the concept of success or failure. However, I/O can fail for a number reason, e.g. if the file is not found or the user does not have read permission. The `Result` representation has a one-to-one mapping because `io::Error` has a separate value for each I/O error, while the `false` value must represent every I/O error.
