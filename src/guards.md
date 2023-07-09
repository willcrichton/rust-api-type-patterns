# Guards

Recall that witnesses are proofs of properties, like being a website admin. In the witness pattern, the actual value (e.g. `admin: Admin`) may not actually be used at runtime &mdash; its presence is purely for statically enforcing access control. When using witnesses, the API designer must carefully ensure that every time a piece of code makes an assumption (e.g. user is an admin), that the code has access to a corresponding witness.

An alternative approach is to **associate a witness with its runtime capabilities.** To dig into that statement, we'll use the running example of a `Mutex<T>` that protects a value of type `T` given a `SystemMutex` (e.g. [pthread_mutex](http://www.cs.kent.edu/~ruttan/sysprog/lectures/multi-thread/pthread_mutex_init.html)). Recall that a mutex should satisfy the following design goals:
1. **Access control:** Only one thread can access `T` at any given time by calling `SystemMutex::lock`.
2. **Cleanup:** Once a thread has finished accessing `T`, it should call `SystemMutex::unlock`.

Here's a hypothetical `Mutex<T>` API:

```rust,ignore
struct Mutex<T> {
  t: UnsafeCell<T>,
  system_mutex: SystemMutex
}

impl<T> Mutex<T> {
  pub fn new(t: T) -> Self { /* .. */ }

  pub fn lock(&self) -> &mut T {
    self.system_mutex.lock();
    unsafe { &mut *self.t.get() }
  }

  pub fn unlock(&self) {
    self.system_mutex.unlock();
  }
}

fn main() {
  let mtx = Mutex::new(1);
  let n = mtx.lock();
  mtx.unlock();

  let n2 = mtx.lock();

  // two mutable pointers to same data! very bad!
  *n = 1;
  *n2 = 2;
}
```

This API nominally enforces access control via encapsulation: the only way to access `T` is by calling `lock`, which ensures that `SystemMutex::lock` is called. However, the issue is that there is no mechanism which relates the lifetime of `&mut T` to the lifetime of the lock on the system mutex. So an API client could forget to call `Mutex::unlock` which violates the cleanup design goal. Or worse, an API client could call `Mutex::unlock` _too early_ which would allow other threads to access `T` simultaneously, violating the access control design goal.


## Witness approach

How could we try to solve this problem with a witness? One possible approach is to return a witness to a mutex being locked, and use that witness to mediate access and cleanup.

```rust,ignore
struct MutexIsLocked;

struct Mutex<T> { /* .. */ }

impl<T> Mutex<T> {
  fn lock(&self) -> MutexIsLocked {
    self.system_mutex.lock();
  }

  fn unlock(&self, _witness: MutexIsLocked) {
    self.system_mutex.unlock();
  }

  fn get(&self, _witness: &MutexIsLocked) -> &mut T {
    unsafe { &mut *self.t.get() }
  }
}

fn main() {
  let mtx = Mutex::new(1);
  let witness = mtx.lock();
  let n = mtx.get(&witness);
  mtx.unlock(witness);

  let witness2 = mtx.lock();
  let n2 = mtx.get(&witness2);
  *n = 1;
  *n2 = 2;
}
```

This interface is better than the previous one because:
1. `unlock` cannot be called at any time, it can only be called after `lock` because the witness ensures that the mutex is locked.
2. `unlock` cannot be called multiple times, because `unlock` takes ownership of the witness.
3. `Mutex::get` cannot be called after `Mutex::unlock` because `unlock` takes ownership of the witness.

However, this interface still does not satisfy our design goals of access control and cleanup.

1. As shown in `main`, the lifetime of the `&mut T` can be longer than the lifetime of the witness. So access control is not enforced because dangling pointers to `T` are held after `unlock`.
2. Nothing relates the `MutexIsLocked` witness to the mutex it came from. You could create two unrelated mutexes, and pass the witness from one into the other.

Again, our main idea returns: **consistency between related elements.** Our witness-based API does not ensure that access to the mutex's data is consistent with access to the witness of the locked mutex.

## Guard approach

Instead of a `MutexIsLocked` witness, we will use a **guard**: a data structure that both proves a property (a mutex is locked) _and_ mediates access to data (the mutex's `T`). The idea is that calling `Mutex::lock` will return a `MutexGuard` which manages the system mutex in its constructor and destructor, and it provides access to the inner `T`.

```rust,ignore
struct MutexGuard<'a, T> {
  lock: &'a Mutex<T>
}

impl<'a, T> MutexGuard<'a, T> {
  fn new(lock: &'a Mutex<T>) -> Self {
    lock.system_mutex.lock();
    MutexGuard { lock }
  }

  fn get(&mut self) -> &mut T {
    &mut *self.lock.t
  }
}

impl<'a, T> Drop for MutexGuard<'a, T> {
  fn drop(&mut self) {
    self.lock.system_mutex.unlock();
  }
}

struct Mutex<T> { /* same as before */ }

impl<T> Mutex<T> {
  pub fn lock<'a>(&'a self) -> MutexGuard<'a, T> {
    MutexGuard::new(self)
  }
}

fn main() {
  let mtx = Mutex::new(1);
  {
    let guard = mtx.lock();
    let n = guard.get();
    *n = 1;
  }

  {
    let guard = mtx.lock();
    let n = guard.get();
    *n = 2;
  }
}
```

This API now enforces both access control and cleanup:
1. **Access control:** when a thread calls `MutexGuard::get`, the borrow on `T` can only last as long as the `MutexGuard`, preventing dangling pointers. And the mutex is guaranteed to be locked by construction when `MutexGuard` exists.
2. **Cleanup:** only after all borrows to `T` have ended will `MutexGuard` be dropped, which then automatically unlocks the system mutex. The API client cannot possibly forget or unlock in the wrong order.

For more examples, this pattern is used throughout the standard library: [`Mutex<T>`](https://doc.rust-lang.org/stable/std/sync/struct.Mutex.html), [`RwLock<T>`](https://doc.rust-lang.org/stable/std/sync/struct.RwLock.html), and [`RefCell<T>`](https://doc.rust-lang.org/stable/std/cell/struct.RefCell.html).


## Closure guards

A variant on the guard design uses closures rather than guard structs. For example, a closure-based mutex:

```rust
struct Mutex<T> { /* same as before */ }

impl<T> Mutex<T> {
  pub fn lock(&self, f: impl FnMut(&mut T)) {
    self.system_mutex.lock();
    f(&mut self.t.get());
    self.system_mutex.unlock();
  }
}

fn main() {
  let mtx = Mutex::new(1);
  mtx.lock(|n| {
    *n = 1;
  });

  mtx.lock(|n| {
    *n = 2;
  });
}
```

The use of closures enables an API to:
1. Execute code at the right time (after the lock is taken)
2. Pass the protected data to the code for a limited duration (`&mut T`)
3. Regain control and cleanup after execution (unlocking the lock)
