# Introduction

Welcome to the `godot-rust` book! This is a work-in-progress user guide for the Rust bindings to GDNative.

If you're new to `godot-rust`, try the [Getting Started](./getting-started.md) tutorial first!

For more information about architecture with `godot-rust`, the [GDNative Overview](gdnative-overview.md) gives a broad overview for _how_ `godot-rust` can be used with a few different use-cases as well as some indepth information for the underlying API.

If have specific code questions that are not covered in the Getting Started guide, please check out the [Frequently Asked Questions](faq.md) or [Recipes](recipes.md) for some additional resources related to configuring `godot-rust`.

If you're new to Rust, before getting started, it is highly recommended that you familiarize yourself with all of the concepts outlined in the officially maintained [Rust Book](https://doc.rust-lang.org/book/) before you getting started with `godot-rust`.

If you have used earlier versions of `godot-rust` before, see [Migrating from godot-rust 0.8](advanced-guides/migrating-0-8.md) for a quick guided tour to the new API.

## About godot-rust

This project specifically supports the [Rust Programming Language](https://www.rust-lang.org/) bindings to the [GDNative API](https://docs.godotengine.org/en/stable/tutorials/plugins/gdnative/gdnative-cpp-example.html) for the Godot Game Engine.

Outside of personal preference, Rust may be a good choice for your game for some of the following reasons:
- Native levels of performance.
- Memory safety validated at compile time.*
- [Fearless Concurrency](https://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html).*
- The leveraging the [cargo](https://doc.rust-lang.org/stable/cargo/) build system to build the GDNative library. 
- The ability to leverage Rust's crate ecosystem from [crates.io](https://crates.io/). 

*: Compile time memory and thread safety guarantees only apply to the Rust code. As the user is allowed to call into the Godot engine (C++ code, via GDNative Foreign Function Interface) or into user-defined scripts (GDScript), some of the validity checks are outside godot-rust's control. However, `godot-rust` guides the user by making clear which operations are potentially unsafe.

For more information about the `godot-rust` project and how it may be useful to you, please refer to the [FAQ Section](faq/meta.md) for project specific information.

## Contributing

The source repository for this book is [hosted on GitHub](https://github.com/godot-rust/book).

## License

The `godot-rust` bindings, and this user guide, are licensed under the [MIT license](LICENSE.md).