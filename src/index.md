# Introduction

Welcome to the **gdnative book**! This is a user guide for the Rust bindings to Godot 3.

Note that `gdnative` is not as actively maintained anymore as its successor, `gdext`. If you are interested in Godot 4, check out the [gdext book](../book).

If you're new to Rust, before getting started, it is highly recommended that you familiarize yourself with concepts outlined in the officially maintained [Rust Book](https://doc.rust-lang.org/book/) before you getting started with godot-rust.


## Navigating this chapter

If you're new to `godot-rust`, try the [Getting Started](intro/index.md) tutorial first!

For more information about architecture with `godot-rust`, the [GDNative Overview](overview/index.md) gives a broad overview of _how_ the library can be used with different use-cases, as well as in-depth information for the underlying API.

If you have specific code questions that are not covered in the _Getting Started_ guide, please check out the [Frequently Asked Questions](faq) or [Recipes](recipes/index.md) for some additional resources related to configuring `godot-rust`.

In case you are coming from earlier versions of `godot-rust` and would like to update, you can have a look at the [Advanced Guides](advanced) chapter for migration guides.


## About godot-rust

This project specifically supports the [Rust Programming Language](https://www.rust-lang.org/) bindings to both [GDNative] and [GDExtension] APIs, for the Godot engine versions 3 and 4, respectively.

Outside of personal preference, Rust may be a good choice for your game for the following reasons:
- Native levels of performance.
- Memory safety validated at compile time.*
- [Fearless Concurrency](https://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html).*
- The [cargo](https://doc.rust-lang.org/stable/cargo/) build system and dependency management.
- The ability to leverage Rust's crate ecosystem from [crates.io](https://crates.io/).

*: Compile time memory and thread safety guarantees only apply to the Rust code. As the user is allowed to call into the Godot engine (C++ code, via GDNative Foreign Function Interface) or into user-defined scripts (GDScript), some of the validity checks are outside godot-rust's control. However, godot-rust guides the user by making clear which operations are potentially unsafe.


## Terminology

To avoid confusion, here is an explanation of names and technologies used within the book.

* [**GDNative**]: C API provided by Godot 3.
* [**GDExtension**]: C API provided by Godot 4.
* **godot-rust**: The entire project, encompassing Rust bindings for Godot 3 and 4, as well as related efforts (book, community, etc.).
* **gdnative** (lowercase): the Rust binding for GDNative (Godot 3).
* **gdext** (lowercase): the Rust binding for GDExtension (Godot 4).
* **Extension**: An extension is a C library developed using gdext. It can be loaded by Godot 4.


## Showcase

If you would like to know about games and other projects in which godot-rust has been employed, check out the [Projects](projects) chapter. At the moment, this is mostly referring to projects built with the Godot 3 bindings, due to their maturity.


## Contributing

The source repository for this book is [hosted on GitHub](https://github.com/godot-rust/book).


## License

The GDNative bindings and this user guide are licensed under the MIT license.  
The GDExtension bindings are licensed under the [Mozilla Public License 2.0](https://www.mozilla.org/en-US/MPL).


[**GDNative**]: https://docs.godotengine.org/en/3.5/tutorials/scripting/gdnative/what_is_gdnative.html
[**GDExtension**]: https://docs.godotengine.org/en/stable/tutorials/scripting/gdextension/what_is_gdextension.html
