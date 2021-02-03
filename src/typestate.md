# Typestate

Let's say we're implementing an API for a read-only file. It should adhere to the following state machine:

<center>
<svg width="562" height="195" version="1.1" xmlns="http://www.w3.org/2000/svg">
	<ellipse stroke="black" stroke-width="1" fill="none" cx="194.5" cy="110.5" rx="38" ry="38"></ellipse>
	<text x="164.5" y="116.5" font-family="Menlo, Courier New" font-size="14">reading</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="355.5" cy="110.5" rx="38" ry="38"></ellipse>
	<text x="342.5" y="116.5" font-family="Menlo, Courier New" font-size="14">eof</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="522.5" cy="110.5" rx="38" ry="38"></ellipse>
	<text x="496.5" y="116.5" font-family="Menlo, Courier New" font-size="14">closed</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="522.5" cy="110.5" rx="32" ry="32"></ellipse>
	<polygon stroke="black" stroke-width="1" points="93.5,110.5 156.5,110.5"></polygon>
	<text x="1.5" y="116.5" font-family="Menlo, Courier New" font-size="14">open(path)</text>
	<polygon fill="black" stroke-width="1" points="156.5,110.5 148.5,105.5 148.5,115.5"></polygon>
	<path stroke="black" stroke-width="1" fill="none" d="M 177.748,76.557 A 28.5,28.5 0 1 1 211.252,76.557"></path>
	<text x="170.5" y="16.5" font-family="Menlo, Courier New" font-size="14">read()</text>
	<polygon fill="black" stroke-width="1" points="211.252,76.557 219.999,73.024 211.909,67.146"></polygon>
	<polygon stroke="black" stroke-width="1" points="232.5,110.5 317.5,110.5"></polygon>
	<polygon fill="black" stroke-width="1" points="317.5,110.5 309.5,105.5 309.5,115.5"></polygon>
	<text x="251.5" y="101.5" font-family="Menlo, Courier New" font-size="14">read()</text>
	<polygon stroke="black" stroke-width="1" points="393.5,110.5 484.5,110.5"></polygon>
	<polygon fill="black" stroke-width="1" points="484.5,110.5 476.5,105.5 476.5,115.5"></polygon>
	<text x="411.5" y="101.5" font-family="Menlo, Courier New" font-size="14">close()</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 491.036,131.749 A 265.6,265.6 0 0 1 225.964,131.749"></path>
	<polygon fill="black" stroke-width="1" points="491.036,131.749 481.608,131.408 486.598,140.074"></polygon>
	<text x="331.5" y="188.5" font-family="Menlo, Courier New" font-size="14">close()</text>
</svg>
</center>

A file is opened into a reading state, and is read until reaching the end-of-file (eof). In either state, the file can be closed. From the lens of access control, our API design goal is to only allow access to certain operations (eg `read()`) when an object is in a corresponding state (eg `reading`).

As a first attempt, here's a perfectly reasonable API that's similar to [`std::fs::File`](https://doc.rust-lang.org/std/fs/struct.File.html):

```rust,ignore
struct File {
  reached_eof: bool,
  /* .. */
}

impl File {
  pub fn open(path: String) -> Option<File> { /* .. */ }

  // Returns None if reached EOF
  pub fn read(&mut self) -> Option<Vec<u8>> {
    if self.reached_eof {
      None
    } else {
      // read the file ..
    }
  }

  pub fn close(self);
}

fn main() {
  let mut f = File::open("test.txt".to_owned()).unwrap();
  while let Some(bytes) = f.read() {
    println!("{:?}", bytes);
  }
  f.read(); // This works! Just will return None
  f.close();
}
```

The key question for this API: how does it prevent calling `read()` in an EOF state? Here, the answer is: it doesn't. Rather, calling `read()` after EOF is a **runtime error**, represented by the `None` branch of the `Option` type. The state machine is only contained internally (`reached_eof`).

## Representing states as structs

Let's say the goal is to prevent the user from _ever_ calling `read()` in an EOF state. That is, the call to `f.read()` after the `while let` _should not compile in the first place_. Now we arrive at the concept of **typestate**: encoding the states of the state machine into the type system.

```rust,ignore
// 1. Each state is its own struct
struct ReadingFile { inner: File }
struct EofFile { inner: File }

enum ReadResult {
  Read(ReadingFile, Vec<u8>),
  Eof(EofFile)
}

impl ReadingFile {
  pub fn open(path: String) -> Option<ReadingFile> { /* .. */ }

  // 2. Calling `read()` takes ownership of ReadingFile
  pub fn read(self) -> ReadResult {
    match self.inner.read() {
      // 3. Access to `ReadingFile` is only given back if not at EOF
      Some(bytes) => ReadResult::Read(self, bytes)
      None => ReadResult::Eof(EofFile { inner: self.inner })
    }
  }

  pub fn close(self) {
    self.inner.close();
  }
}

impl EofFile {
  pub fn close(self) {
    self.inner.close();
  }
}

fn main() {
  let mut file = ReadingFile::open("test.txt".to_owned()).unwrap();
  loop {
    match file.read() {
      ReadResult::Read(f, bytes) => {
        println!("{:?}", bytes);
        file = f;
      }
      ReadResult::Eof(f) => {
        f.close();
        break;
      }
    }
  }
  // file has been moved, can't call file.read() here
}
```

This API design has three key ideas:
1. **Each state is a struct**: the `ReadingFile` and `EofFile` states are represented as distinct structs. This way, we can associate different methods with each.
2. **Each state has only its transitions implemented:** the `read()` method is implemented for `ReadingFile`, but not for `EofFile`, while both can call `close()`.
3. **Transitions between states consume ownership:** the `read()` method consumes `ReadingFile` and returns whichever state comes after. This prevents calling methods on an old state.

## Moving states into a type parameter

One ugliness in this design is that the user has to manage a single logical object (the file) but split into multiple actual types (the states). An alternative design is to represent states as a **type parameter** to a single object, like so:

```rust,ignore
struct Reading;
struct Eof;

struct File2<State> {
  inner: File,
  _state: PhantomData<State>
}

enum ReadResult {
  Read(File2<Reading>, Vec<u8>),
  Eof(File2<Eof>)
}

impl File2<Reading> {
  pub fn open(path: String) -> Option<File2<Reading>> { /* .. */ }

  pub fn read(self) -> ReadResult {
    match self.inner.read() {
      // 3. Access to `ReadingFile` is only given back if not at EOF
      Some(bytes) => ReadResult::Read(self, bytes)
      None => ReadResult::Eof(File2 { inner: self.inner, _state: PhantomData })
    }
  }

  pub fn close(self) {
    self.inner.close();
  }
}

impl File2<Eof> {
  pub fn close(self) {
    self.inner.close();
  }
}
```

The API is largely the same, and the usage example from before still works. But on the implementation side, we now have to carry around this [`PhantomData`](https://doc.rust-lang.org/std/marker/struct.PhantomData.html) marker or else Rust complains about an unused type parameter.

## Further reading

* [Stanford CS 242 - Typestate](https://stanford-cs242.github.io/f19/lectures/08-2-typestate)
* [Typestates in Rust](https://yoric.github.io/post/rust-typestate/)
* [A hammer you can only hold by the handle](https://blog.systems.ethz.ch/blog/2018/a-hammer-you-can-only-hold-by-the-handle.html)
