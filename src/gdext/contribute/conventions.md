# Code and API conventions


<!-- toc -->

## Bikeshed auto-painting

In general, we try to automate as much as possible during CI. This ensures a consistent code style and avoids unnecessary work during
pull request reviews.

In particular, we use the following tools:

* [**rustfmt**] for code formatting ([config options]).
* [**clippy**] for lints and style warnings ([list of lints]).
* Clang's [**AddressSanitizer**] and [**LeakSanitizer**] for memory safety.
* Various specialized tools:
  * [**skywalking-eyes**] to enforce license headers.
  * [**cargo-deny**] and [**cargo-machete**] for dependency verification.

In addition, we have unit tests (`#[test]`), doctests and Godot integration tests (`#[itest]`). 
See [Dev tools] for more information.

[**rustfmt**]: https://github.com/rust-lang/rustfmt
[config options]: https://rust-lang.github.io/rustfmt
[**clippy**]: https://doc.rust-lang.org/stable/clippy/usage.html
[list of lints]: https://rust-lang.github.io/rust-clippy/master/index.html
[**AddressSanitizer**]: https://clang.llvm.org/docs/AddressSanitizer.html 
[**LeakSanitizer**]: https://clang.llvm.org/docs/LeakSanitizer.html
[**skywalking-eyes**]: https://github.com/apache/skywalking-eyes
[**cargo-deny**]: https://embarkstudios.github.io/cargo-deny
[**cargo-machete**]: https://github.com/bnjbvr/cargo-machete
[Dev tools]: dev-tools.md


## API design principles

Different Rust libraries have different goals, which is reflected in API design choices. gdext is written in Rust, but interoperates with 
a C++/GDScript game engine, which means that we sometimes need to go unconventional ways to achieve good user experiences.
This may sometimes be surprising for Rust developers.

We envision the following core principles as a guideline for API design:

1. **Simplicity**  
   Prefer self-explanatory, straightforward interfaces.
   * Avoid abstractions that don't add value to the user. 
     Do not over-engineer prematurely just because it's possible; follow [YAGNI]. Avoid [premature optimization].
   * Examples to avoid: traits that are not used polymorphically, type-state pattern, many generic parameters,
     layers of wrapper types/functions that simply delegate logic.
   * Sometimes, runtime errors are better than compile-time errors. Most users are building a game, where fast iteration is key.
     Use `Option`/`Result` when errors are recoverable, and panics when the user must fix their code. See also [Ergonomics and panics].

2. **Maintainability**  
   Every line of code added **must be maintained, potentially indefinitely**.
   * Consider that it may not be you working with it in the future, but another contributor or maintainer, maybe a year from now.
   * Try to see the bigger picture -- how important is a specific in the overall library? How much detail is necessary?
     Balance the amount of code with its real-world impact for users.
   * Document non-trivial thought processes and design choices as inline `//` comments.
   * Document behavior, invariants and limitations in `///` doc comments.

3. **Consistency**  
   As a user, having a uniform experience when using different parts of the library is important.
   This reduces the cognitive load of learning and using the library, requires less doc lookup and makes users more efficient.
   * Look at existing code and try to understand its patterns and conventions.
   * Before doing larger refactorings or changes of existing systems, try to understand the underlying design choices.

See these as guidelines, not hard rules. If you are unsure, please don't hesitate to ask questions and discuss different ideas.
We highly appreciate if people come up with a rough design before spending a large amount of time on implementation; this helps
save time on approaches that may not work.


## Technicalities

This section lists specific style conventions that have caused some confusion in the past.
Following them is nice for consistency, but it's not the top priority of this project. Hopefully, we can automate some of them over time.


### Formatting

`rustfmt` is the authority on formatting decisions. If there are good reasons to deviate from it, e.g. data-driven tables in tests,
use `#[rustfmt::skip]`. rustfmt does not work very well with macro invocations, but such code should still follow `rustfmt`'s
formatting choices where possible.

Line width is 120-145 characters (mostly relevant for comments).  
We use separators starting with  `// ---` to visually divide sections of related code.


### Code organization

1. Anything that is not intended to be accessible by the user, but must be `pub` for technical reasons, should be marked as `#[doc(hidden)]`.
   * This does [**not** constitute part of the public API][public-api]. 

1. We do not use the `prelude` inside the project, except in examples and doctests.

1. Inside `impl` blocks, we _roughly_ try to follow the order:
   * Type aliases in traits (`type`) 
   * Constants (`const`)
   * Constructors and associated functions
   * Public methods
   * Private methods (`pub(crate)`, private, `#[doc(hidden)]`)

1. Inside files, there is no strict order yet, except `use` and `mod` at the top. Prefer to declare public-facing symbols before private ones.


### Types

1. Avoid tuple-enums `enum E { Var(u32, u32) }` and tuple-structs `struct S(u32, u32)` with more than 1 field. Use named fields instead.

1. Derive order is `#[derive(GdextTrait, ExternTrait, Default, Copy, Clone, Eq, PartialEq, Ord, PartialOrd, Hash, Debug)]`.
   * `GdextTrait` is a custom derive defined by gdext itself (in any of the crates).
   * `ExternTrait` is a custom derive by a third-party crate, e.g. `nanoserde`.
   * The standard traits follow order _construction, comparison, hashing, debug display_.
     More expressive ones (`Copy`, `Eq`) precede their implied counterparts (`Clone`, `PartialEq`).


### Functions

1. Getters don't have a `get_` prefix.

1. Use `self` instead of `&self` for `Copy` types, unless they are really big (such as `Transform3D`).

1. For `Copy` types, avoid in-place mutation `vector.normalize()` and prefer `vector = vector.normalized()`.
 
1. Annotate with `#[must_use]` when ignoring the return value is likely an error. Example: builder APIs.


### Proc-macro attributes

Concerns both `#[proc_macro_attribute]` and the attributes attached to a `#[proc_macro_derive]`.

1. Attributes always have the same syntax: `#[attr(key = "value", key2, key_three = 20)]`
   * `attr` is the outer name grouping different key-value pairs in parentheses.  
     A symbol can have multiple attributes, but they cannot share the same name.
   * `key = value` is a key-value pair. just `key` is a key-value pair without a value.
     * Keys are always `snake_case` identifiers.  
     * Values are typically strings or numbers, but can be more complex expressions.
     * Multiple key-value pairs are separated by commas. Trailing commas are allowed.

2. In particular, avoid these forms:
   * `#[attr = "value"]` (top-level assignment)
   * `#[attr("value")]` (no key -- note that `#[attr(key)]` is allowed)
   * `#[attr(key(value))]`
   * `#[attr(key = value, key = value)]` (repeated keys)

The reason for this choice is that each attribute maps nicely to a map, where values can have different types.
This allows for a recognizable and consistent syntax across all proc-macro APIs. Implementation-wise, this pattern is
directly supported by the `KvParser` type in gdext, which makes it easy to parse and interpret attributes.


[YAGNI]: https://en.wikipedia.org/wiki/YAGNI
[premature optimization]: https://en.wikipedia.org/wiki/Program_optimization#When_to_optimize
[public-api]: https://godot-rust.github.io/docs/gdext/master/godot/#public-api
[Ergonomics and panics]: https://godot-rust.github.io/docs/gdext/master/godot/#ergonomics-and-panics
