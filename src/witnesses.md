# Witnesses

Let's say we're designing a website with a panel that only admins can access. A naive approach relies on enforcing access control by convention:

```rust,ignore
/// IMPORTANT: Only call this function when logged in as admin!
fn render_admin_panel() -> Html {
  // ...
}

fn admin_panel_route_ok() -> Html {
  // We could remember to check...
  if current_user().is_admin() {
    render_admin_panel();
  } else {
    render_404();
  }
}

fn admin_panel_route_whoops() -> Html {
  // Or we could just forget!
  render_admin_panel();
}
```

Here's an alternative API design that turns the convention into a requirement:

```rust,ignore
mod admin {
  // When Admin is hidden in a module,
  // it can only be constructed by exported functions
  pub struct Admin { /* .. */ }

  impl User {
    // Only returns Some(Admin) if user is an admin
    pub fn try_admin(&self) -> Option<Admin> { /* ... */ }
  }
}

// We add Admin as a parameter to the render function
fn render_admin_panel(_admin: admin::Admin) -> Html {
  // ...
}

fn admin_panel_route_ok() -> Html {
  if let Some(admin) = current_user().try_admin() {
    render_admin_panel(admin);
  } else {
    render_404();
  }
}

fn admin_panel_route_whoops() -> Html {
  // TYPE ERROR: current_user() is not an admin!
  render_admin_panel(current_user());

  // TYPE ERROR: Admin has private fields and cannot be constructed
  render_admin_panel(admin::Admin { .. });
}
```

Here, the `Admin` struct is an example of a **witness**: an object that proves (or "witnesses") a particular property, e.g. that a user is an admin. (The term comes from [proof theory](https://en.wikipedia.org/wiki/Witness_(mathematics)).) The witness pattern relies on two ideas:

1. **Witnesses prove properties by construction.** The only way to create a value of type `Admin` should be via `try_admin`. If this is true, then having access to a value of type `Admin` is a proof that a user is an admin. "By construction" means the act of creating a value is equivalent to creating a proof of what that value represents (see: [Curry-Howard correspondence](https://en.wikipedia.org/wiki/Curry%E2%80%93Howard_correspondence)).

2. **Code that relies on properties takes witnesses as input.** If an admin panel should only be rendered by an admin, then `render_admin_panel` should require that the function caller prove it has admin access. Even if the actual `admin` value is never used, because it is a required input, the function cannot be called without proof of access.

With these two ideas, the witness pattern can statically enforce access control. Witnesses encode access capabilities, and actions require witnesses before being executed. For a real-world example, check out [Request Guards](https://rocket.rs/v0.4/guide/requests/#request-guards) in the Rocket framework (see also [Sergio's lecture notes](https://stanford-cs242.github.io/f18/lectures/07-1-sergio.html#rocket)).
