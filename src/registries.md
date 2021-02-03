# Registries

Extensible systems, or frameworks where the client plugs in their own functionality, often have a notion of "registration" to associate the user's code with a piece of the framework. For example:

* In event systems, users can register callbacks to be triggered when an event occurs.
* In dependency injection systems, users register dependencies which gets plumbed to other code that is registered to request that dependency.

These systems use **registries** to maintain sets of event listeners or dependencies. The key API design challenge is to maintain consistency between the registered objects and the code that reads the registry. As a running example, we will use an event system. Consider this API that works like JavaScript's [`addEventListener`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener):

```rust
# use std::any::Any;
# use std::collections::HashMap;
type EventListener = Box<dyn Fn(&dyn Any)>;

#[derive(Default)]
struct EventRegistry {
  listeners: HashMap<String, Vec<EventListener>>
}

impl EventRegistry {
  fn add_event_listener(&mut self, event: String, f: EventListener) {
    self.listeners.entry(event).or_insert_with(Vec::new).push(f);
  }

  fn trigger(&self, event: String, data: &dyn Any) {
    let listeners = self.listeners.get(&event).unwrap();
    for listener in listeners.iter() {
      listener(data);
    }
  }
}

struct OnClick { mouse_x: f32, mouse_y: f32 }

fn main() {
  let mut events = EventRegistry::default();
  events.add_event_listener("click".to_owned(), Box::new(|event| {
    let event = event.downcast_ref::<OnClick>().unwrap();
    assert_eq!(event.mouse_x, 1.);
  }));

  let event = OnClick { mouse_x: 1., mouse_y: 3. };
  events.trigger("click".to_owned(), &event);
}
```

How could a user incorrectlmouse_y work with this API? They could:

* **Typo the event name**: as with all stringly-typed programming, the event name could be the wrong string, like `"clack"` instead of `"click"`, for either the listener or the trigger.

* **Send the wrong payload for an event**: event listeners take literally any type as input. A user could trigger a `MouseEvent` for the `"click"` string.

* **Use the event payload incorrectly:**: a user could incorrectly cast the payload to `MouseEvent` for a listener to the `"click"` event.

The core issue is that the event name `"click"` and the event data `OnClick` are linked by convention but not by design. This issue is inherent to registries that use strings to identify objects.

Instead, I will show you how to design both an event system and a dependency injection system using **types as keys intead of strings as keys**. Type-based registries will avoid all of the issues in the API above.

## 1. Type-safe events

Our goal is to write an API that looks like this from the client's perspective:

```rust,ignore
struct OnClick {
  mouse_x: f32,
  mouse_y: f32,
}

let mut events = EventRegistry::new();

// The event is passed in as a type parameter instead of a string
events.add_event_listener::<OnClick>(|event| {
  assert_eq!(event.mouse_x, 10.);
  assert_eq!(event.mouse_y, 5.);
});

// The event type could alternatively be inferred from the closure
events.add_event_listener(|event: &OnClick| {
  assert_eq!(event.mouse_x, 10.);
  assert_eq!(event.mouse_y, 5.);
});

events.trigger(&OnClick {
  mouse_x: 10.,
  mouse_y: 5.,
})
```

The outline of the underlying API looks like this:

```rust, ignore
trait Event = 'static;
trait EventListener<E> = Fn(&E) -> () + 'static;

struct EventRegistry { /* .. */ };

impl EventRegistry {
  fn add_event_listener<E: Event>(
    &mut self,
    f: impl EventListener<E>
  ) {
    /* .. */
  }

  fn trigger<E: Event>(
    &self,
    event: &E
  ) {
    /* .. */
  }
}
```

Each method on the event registry takes an event type `E` as input, rather than a string key. The registry associates listeners with that type, and then looks up listeners with that type.

### Mapping types to values

To associate values (e.g. event listeners) with types, we will make a core building block: the `TypeMap`.

Rust has [`std::any`](https://doc.rust-lang.org/std/any/) for this purpose. [`TypeId`](https://doc.rust-lang.org/std/any/struct.TypeId.html) allows us to get a unique, hashable identifier for each type. [`Any`](https://doc.rust-lang.org/std/any/trait.Any.html) allows us to up-cast/down-cast objects at runtime. Hence, our `TypeMap` will map from `TypeId` to `Box<dyn Any>`.

```rust,ignore
use std::collections::HashMap;
use std::any::{TypeId, Any};

struct TypeMap(HashMap<TypeId, Box<dyn Any>>);
```

To add an element to the map:

```rust,ignore
impl TypeMap {
  pub fn set<T: Any + 'static>(&mut self, t: T) {
    self.0.insert(TypeId::of::<T>(), Box::new(t));
  }
}
```

This means our map has one unique value for a given type. For example, if we use the `TypeMap` like this:

```rust,ignore
let mut map = TypeMap::new();
map.set::<i32>(1);
```

> Aside: the syntax `::<i32>` is Rust's "turbofish". It explicitly binds a type parameter of a polymorphic function, rather than leaving it to be inferred. [Further](https://stackoverflow.com/questions/52360464/what-is-the-syntax-instance-methodsomething/52361559) [explanation](https://matematikaadit.github.io/posts/rust-turbofish.html) [here](https://techblog.tonsser.com/posts/what-is-rusts-turbofish).

Then we insert a value `1` at the key `TypeId::of::<i32>()`. We can also implement `has` and `get` functions:

```rust,ignore
impl TypeMap {
  pub fn has<T: Any + 'static>(&self) -> bool {
    self.0.contains_key(&TypeId::of::<T>())
  }

  pub fn get_mut<T: Any + 'static>(&mut self) -> Option<&mut T> {
    self.0.get_mut(&TypeId::of::<T>()).map(|t| {
      t.downcast_mut::<T>().unwrap()
    })
  }
}
```

Look carefully at `get_mut`. The inner hash map returns a value of type `Box<dyn Any>`, which we can `downcast_mut` to become a value of type `&mut T`. This operation is guaranteed to not fail, because only values of type `T` are stored in the hash map under the key for `T`.

### Implementing the event system

With the `TypeMap` in hand, we can finish our event system. For the `EventRegistry`, the `TypeMap` will map from events to a vector of listeners.

```rust,ignore
struct EventDispatcher(TypeMap);
type ListenerVec<E> = Vec<Box<dyn EventListener<E>>>;

impl EventDispatcher {
  fn add_event_listener<E>(&mut self, f: impl EventListener<E>) {
    if !self.0.has::<ListenerVec<E>>() {
      self.0.set::<ListenerVec<E>>(Vec::new());
    }

    let listeners = self.0.get_mut::<ListenerVec<E>>().unwrap();
    listeners.push(Box::new(f));
  }

  fn trigger<E>(&self, event: &E) {
    if let Some(listeners) = self.0.get::<ListenerVec<E>>() {
      for callback in listeners {
        callback(event);
      }
    }
  }
}
```

Returning to our API design goals, this design enforces consistency between related elements: an event listener's payload must match the event it's registered to.

## 2. Type-safe dependency injection

We'll walk through another example of using type registries to avoid unsafe string-based design. The basic idea of dependency injection (DI) is that you have a component that depends on another, like a web server using a database. However, you don't want to hard-code a particular database constructor, and rather make it easy to swap in different databases. For example:

```rust,ignore
trait Database {
  fn name(&self) -> &'static str;
}

struct MySQL;
impl Database for MySQL {
  fn name(&self) -> &'static str { "MySQL" }
}

struct Postgres;
impl Database for Postgres {
  fn name(&self) -> &'static str { "Postgres" }
}

struct WebServer { db: Box<dyn Database> }
impl WebServer {
  fn run(&self) {
    println!("Db name: {}", self.db.name());
  }
}
```

To implement DI, we need two things:
* We need a way to register a global `Database` at runtime to a particular instance, e.g. `MySQL` or `Postgres`.
* We need a way to describe a constructor for `WebServer` that fetches the registered `Database` instance.

With these pieces, we can use our DI system like so:

```rust,ignore
let mut manager = DIManager::new();
manager.build::<MySQL>().unwrap();
let server = manager.build::<WebServer>().unwrap();
server.lock().unwrap().run(); // prints Db name: MySQL
```

### DI constructors

First, we'll define a trait `DIBuilder` that represents a constructor within our DI system.

```rust,ignore
trait DIBuilder {
  type Input;
  type Output;

  fn build(input: Self::Input) -> Self::Output;
}
```

The `build` method is a static method (doesn't take `self` as input). It just takes `Input` as input, and produces `Output` as output. The key idea is that because `Input` and `Output` are [associated types](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#specifying-placeholder-types-in-trait-definitions-with-associated-types), we can inspect them later on. We will need to find values for `Input` and to store `Output` in our DI manager.

We implement `DIBuilder` for each type in the system. The databases have no inputs, so their input is `()`. Their return type is `Box<dyn Database>`, meaning they are explicitly cast to a trait object so they can be used interchangeably.

```rust,ignore
impl DIBuilder for MySQL {
  type Input = ();
  type Output = Box<dyn Database>;
  fn build((): ()) -> Box<dyn Database> {
    Box::new(MySQL)
  }
}

impl DIBuilder for Postgres {
  type Input = ();
  type Output = Box<dyn Database>;
  fn build((): ()) -> Box<dyn Database> {
    Box::new(Postgres)
  }
}

impl DIBuilder for WebServer {
  type Input = (Box<dyn Database>,);
  type Output = WebServer;

  fn build((db,): Self::Input) -> WebServer {
    WebServer { db }
  }
}
```

### DI manager

Now that we know the dependency structure of our objects, we need a centralized manager to store the objects and fetch their dependencies.

```rust,ignore
struct DIManager(TypeMap);
```

In this `TypeMap`, we will store the constructed objects. For example, once we make a `Box<dyn Database>`, then we will map `TypeId::of::<Box<dyn Database>>` to one of `Box<Postgres>` or `Box<MySQL>`.

```rust,ignore
impl DIManager {
  fn build<T: DIBuilder>(&mut self) -> Option<T::Output> {
    let input = /* get the inputs, somehow */;
    let obj = T::build(input);
    self.0.set::<T::Output>(obj);
    Some(obj)
  }
}
```

Ignoring how we fetch dependencies for now, this function calls the `DIBuilder::build` implementation for `T`, then stores the result in the `TypeMap`. This approach _almost_ works, except not for Rust: ownership of `obj` is passed into `TypeMap` and the result of `build`.

And, intuitively, this makes sense. If a component like a database cursor needs to be shared across many downstream components, it needs some kind of access protection. Hence, we tweak our interface a bit to wrap everything in an [`Arc`](https://doc.rust-lang.org/std/sync/struct.Arc.html) and [`Mutex`](https://doc.rust-lang.org/std/sync/struct.Mutex.html).

```rust,ignore
type DIObj<T> = Arc<Mutex<T>>;

impl DIManager {
  fn build<T: DIBuilder>(&mut self) -> Option<DIObj<T::Output>> {
    let input = /* get the inputs, somehow */;
    let obj = T::build(input);
    let sync_obj = Arc::new(Mutex::new(obj));
    self.0.set::<DIObj<T::Output>>(sync_obj.clone());
    Some(sync_obj)
  }
}
```

### DI dependencies

Finally, we need a way to implement the `let input` line in `DIManager::build`. Given a type `T: DIBuilder`, it has an associated type `T::Input` that represents the inputs needed to build it.

To simplify the problem, imagine `T::Input = S` where `S: DIBuilder`. Then if `S` has already been built, e.g. `T = WebServer` and `S = Box<dyn Database>`, we can fetch it directly from the typemap:

```rust,ignore
let input = self.0.get::<T::Input>().map(|obj| obj.clone())?;
```

However, in practice `T::Input` could be several dependencies. For example, if our server depends on a configuration, it might be:

```rust,ignore
impl DIBuilder for WebServer {
  type Input = (Box<dyn Database>, Box<dyn ServerConfig>);
  type Output = WebServer;

  fn build((db, config): Self::Input) -> WebServer {
    WebServer { db, config }
  }
}
```

Let's assume now `T::Input` is always a tuple `(S1, S2, ...)` of types where each type `Si : DIBuilder`. Ideally, we could write something like:

```rust,ignore
let input = (T::Input).map(|S| {
  self.0.get::<S>().map(|obj| obj.clone()).unwrap()
});
```

But, alas, our language of expressions is not our language of types. Such a thing is the provenance of languages we can only [dream about](https://leanprover.github.io/). Instead, we have to cleverly use traits to inductively define a way to extract inputs from the tuple. To start, we'll make a trait that gets an object of a particular type from the `DIManager`:

```rust,ignore
trait GetInput: Sized {
  fn get_input(manager: &DIManager) -> Option<Self>;
}
```

For `DIObj<T>`, this means looking up the type in the `TypeMap`:

```rust,ignore
impl<T: 'static> GetInput for DIObj<T> {
  fn get_input(manager: &DIManager) -> Option<Self> {
    manager.0.get::<Self>().map(|obj| obj.clone())
  }
}
```

Then for tuples of `DIObj<T>`, we can make an inductive definition like so:

```rust,ignore
impl GetInput for () {
  fn get_input(_manager: &DIManager) -> Option<Self> {
    Some(())
  }
}

impl<S: GetInput, T: GetInput> GetInput for (S, T) {
  fn get_input(manager: &DIManager) -> Option<Self> {
    S::get_input(manager).and_then(|s| {
      T::get_input(manager).and_then(|t| {
        Some((s, t))
      })
    })
  }
}
```

Then we can modify our `WebServer` example to use an inductive structure:

```rust,ignore
impl DIBuilder for WebServer {
  type Input = (DIObj<Box<dyn Database>>,(DIObj<Box<dyn ServerConfig>>,()));
  type Output = WebServer;

  fn build((db, (config, ())): Self::Input) -> WebServer {
    WebServer { db, config }
  }
}
```

> Aside: in practice, rather than doing nested pairs, you can use a macro to create the `GetInput` impl for many tuple types [like this](https://github.com/amethyst/shred/blob/0.10.2/src/system.rs#L438-L469).

And at last, we can implement `DIManager::build`:

```rust,ignore
impl DIManager {
  pub fn build<T: DIBuilder>(&mut self) -> Option<DIObj<T::Output>> {
    let input = T::Input::get_input(self)?;
    let obj = T::build(input);
    let sync_obj = Arc::new(Mutex::new(obj));
    self.0.set::<DIObj<T::Output>>(sync_obj.clone());
    Some(sync_obj)
  }
}
```

Now, the expression `T::Input::get_input(self)` will convert a tuple of types into a tuple of values of those types.

### Final API example

With the API complete, our example now looks like this:

```rust,ignore
let mut manager = DIManager::new();
manager.build::<MySQL>().unwrap();
let server = manager.build::<WebServer>().unwrap();
server.lock().unwrap().run();
```

We can construct a `WebServer` without explicitly passing in a `dyn Database` instance. When we use the `server`, we have to explicitly call `.lock()` now that it's wrapped in a mutex. And lo, our dependencies have been injected.
