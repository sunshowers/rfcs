- Feature Name: `unknown_non_exhaustive`
- Start Date: 2021-08-17
- RFC PR: TBD
- Rust Issue: TBD

# Summary
[summary]: #summary

This RFC introduces a new lint check called `unknown_non_exhaustive`, which provides a way for new enum variants and
struct fields to be added in a way that:
* avoids breaking changes
* while still producing a lint failure if a particular, known pattern isn't specified.

The new lint check defaults to `allow`.

# Motivation
[motivation]: #motivation

This RFC fleshes out the "not exhaustive enough" lint mentioned as a future possibility in [RFC 2008]. 

## Enums
[motivation-enums]: #motivation-enums

While non-exhaustive types prevent breaking changes, they also disable exhaustiveness checking as a side effect. 

For example, a hypothetical `ErrorKind` enum in a library might look like:

```rust
#[non_exhaustive]
pub enum ErrorKind {
    NotFound,
    Interrupted,
}
```

A downstream user in matching against this enum would have to write:

```rust
match error_kind {
    ErrorKind::NotFound => ...,
    ErrorKind::Interrupted => ...,
    _ => ...,
}
```

If `ErrorKind` gains a new variant in a minor version of the library:

```rust
#[non_exhaustive]
pub enum ErrorKind {
    NotFound,
    Interrupted,
    TimedOut,
}
```

there's currently no way for `rustc` to tell the user that the new `TimedOut` variant exists.
This is in contrast to regular (exhaustive) enums, where a new variant is a breaking change and the
user will receive a compilation failure.

With the new lint check, a user would have the ability to annotate the wildcard pattern:

```rust
#[warn(unknown_non_exhaustive)]
match error_kind {
    // ...
    _ => ...,
}

```

and receive a warning if the new variant hasn't been handled.

The lint can also be applied at the expression, module or crate level:

```rust
#![warn(unknown_non_exhaustive)]

#[allow(unknown_non_exhaustive)]
match error_kind {
    // ...
    _ => ...,
}
```

In some cases, the downstream user may wish to fail compilation on an unknown pattern. In that case, they can `deny` or
`forbid` the lint:

```rust
#[deny(unknown_non_exhaustive)]
match error_kind {
    // ...
    _ => ...,
}
```

## Structs
[motivation-structs]: #motivation-structs

The most common use for non-exhaustive structs is config types with public fields.

For example, borrowing the example in [RFC 2008], a config struct may look like:

```rust
#[non_exhaustive]
pub struct Config {
    pub window_width: u16,
    pub window_height: u16,
}
```

Code that matches the struct must provide a wildcard attribute:

```rust
if let Ok(config { window_width, window_height, .. }) = load_config() {
    // ...
}
```

As with enums, if a new field is added to the struct, there is no way for `rustc` to alert the user
that it has been added.

With this lint, users would be able to warn, or fail compilation on, a missing field.

```rust
#[warn(unknown_non_exhaustive)]
if let Ok(config { window_width, window_height, .. }) = load_config() {
    // the line above would produce a warning if a new field is added 
    // ...
}
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `unknown_non_exhaustive` lint detects when a non-exhaustive enum in an upstream crate has a variant that isn't
handled, or a non-exhaustive enum variant/struct has a new field that isn't handled.

## Example

In an upstream crate:

```rust
#[non_exhaustive]
pub enum ErrorKind {
    NotFound,
    Interrupted,
    TimedOut,
}
```

In your code:

```rust
#[warn(unknown_non_exhaustive)]
match error_kind {
    ErrorKind::NotFound => ...,
    ErrorKind::Interrupted => ...,
    // A wildcard variant is required because this is a non-exhaustive enum.
    _ => ...,
}
```

This will produce a lint warning (exact warning TBD).

## Explanation

The `#[non_exhaustive]` attribute allows upstream libraries to add new variants to an enum, or public fields to an enum
variant/struct, without causing a breaking change. However, some users may wish to receive a warning, or fail to
compile, when a new variant is added in a minor version of a dependency. This lint is issued when an unhandled variant
(or field) is detected in a non-exhaustive enum (or enum variant/struct).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Exact implementation details are TBD, but a general outline:
* `match` and other destructuring expressions currently have an exhaustiveness check.
* For non-exhaustive types, the check would first be executed with wildcard pattern(s): if they doesn't exist,
then compiles would fail as they do today.
* Then, the same check would be executed with wildcard pattern(s) and the `#[non_exhaustive]` annotation both removed.
If this second exhaustiveness check fails, the lint would be fired.

## Corner cases
[corner-cases]: #corner-cases

### Matching against multiple enums at once
[matching-against-multiple-enums-at-once]: #matching-against-multiple-enums-at-once

```rust
match (error_kind_1, error_kind_2) {
    (ErrorKind::NotFound, ErrorKind::NotFound) => ...,
    (ErrorKind::Interrupted, ErrorKind::NotFound) | (ErrorKind::Interrupted, ErrorKind::Interrupted) => ...,
    (ErrorKind::NotFound, _) => ...,
    (_, _) => ...,
}
```

One strategy for handling these would be: in the second exhaustiveness check, remove *all* wildcard patterns
corresponding to non-exhaustive enums. So, for the second check, the `match` becomes

```rust
match (error_kind_1, error_kind_2) {
    (ErrorKind::NotFound, ErrorKind::NotFound) => ...,
    (ErrorKind::Interrupted, ErrorKind::NotFound) | (ErrorKind::Interrupted, ErrorKind::Interrupted) => ...,
}
```

# Drawbacks
[drawbacks]: #drawbacks

One of the drawbacks of this lint is that handling these unknown cases would require bumping the minimum version
of the dependency. For example, if version 1.4.0 of the dependency adds the `TimedOut` variant, the downstream user
would have to raise its minimum requirement to 1.4.0 in order to handle the variant. However, by opting into the lint,
the downstream user may be regarded as expressing a desire to stay relatively up-to-date with the upstream library, so
this may be a net positive.

Another drawback is that upstream libraries cannot force downstream users to use this lint. However, they can recommend
using it in the documentation if there's a compelling reason to.

Note that this lint check, even in `deny` mode, does *not* make `#[non_exhaustive]` a no-op. This is because Cargo caps
lints to `allow` for dependencies, so this won't affect transitive dependents of the downstream user.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

* One option that was considered was making warnings part of the enum declaration:

    ```rust
    #[non_exhaustive(warn)]
    pub enum ErrorKind {
        NotFound,
        Interrupted,
        TimedOut,
    }
    ```

    However, this cedes control of the lint to the upstream library: downstream users cannot choose to have different
behavior. It also raises thorny questions about whether moving an enum from `#[non_exhaustive]` to
`#[non_exhaustive(warn)]` (or back) counts as a breaking change. The proposed solution avoids these issues.

* Another possibility is to make this check an attribute rather than a lint, like in Swift (more on Swift in [Prior art](#prior-art)):

    ```rust
    match error_kind {
        ErrorKind::NotFound => ...,
        ErrorKind::Interrupted => ...,
        #[unknown]
        _ => ...
    }
    ```
  
    However, this check fits more naturally into Rust as a lint; in particular, the lint correctly inherits Cargo's
    capping of lints in dependencies to `allow`.

* Making the lint default to `warn` was considered as well, but doing so would likely cause warnings on
too much existing code. However, it may be worth changing the default behavior in a future edition of Rust.

* The lint is proposed to work on `match` and other destructuring expressions, but not on wildcard patterns themselves.
So users wouldn't be able to write:

    ```rust
    match error_kind {
        ErrorKind::NotFound => ...,
        ErrorKind::Interrupted => ...,
        #[warn(unknown_non_exhaustive)]
        _ => ...
    }
    ```

    This avoids issues with more complex matching situations, such as
    [matching multiple enums](#matching-against-multiple-enums-at-once).

* Making this a `clippy` lint rather than a `rustc` lint was considered. However, exhaustiveness checks are tricky
to implement and `rustc` already has them built-in.

* Not implementing this means that the only way for exhaustiveness checking to work is to drop the `#[non_exhaustive]`
annotation. This is currently [preventing some libraries from reaching a stable
version](https://github.com/rusqlite/rusqlite/issues/783).

# Prior art
[prior-art]: #prior-art

Swift has an [`@unknown` annotation](https://github.com/apple/swift-evolution/blob/main/proposals/0192-non-exhaustive-enums.md)
which is motivated by the same concerns. Swift 5 warns by default if the `@unknown`
annotation is missing, and Swift 4 produces no diagnostic. This proposal aligns with the Swift 4 behavior by default,
but leaves open the possibility of switching to warning by default in a future Rust edition.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What is the most appropriate name for the lint?
- Is this the correct way to handle [matching multiple enums](#matching-against-multiple-enums-at-once)?
- Are there any other corner cases to consider?
- Do the implementation details need to be fleshed out further?

# Future possibilities
[future-possibilities]: #future-possibilities

It might make sense to warn by default in the future, perhaps in the next Rust edition.

In the future, there could be a method by which upstream libraries can provide a default lint behavior per-type, which
downstream consumers can override.

[RFC 2008]: https://github.com/sunshowers/rfcs/blob/master/text/2008-non-exhaustive.md
