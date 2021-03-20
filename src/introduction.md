# Introduction

Welcome to the `godot-rust` book! This is a work-in-progress user guide for the Rust bindings to GDNative.

If you're new to `godot-rust`, try the [Getting Started](./getting-started.md) tutorial first!

If you have used earlier versions of `godot-rust` before, see [Migrating from godot-rust 0.8.x](./migrating-0-8.md) for a quick guided tour to the new API.

## What is godot-rust

The Godot Engine has rich user scripting support. Using a DLL binding system called GDNative you can create "Scripts" in Rust and attach them to Godot Nodes. Everything you can do from Godot in a language such as GDScript you can do from rust using godot-rust.

This DLL - short for Dynamically linked Library (rust calls it a dylib) also has full access to Rust's entire featureset. You can read and write files, open network ports, connect to a custom server, run a custom website and more.

Rust itself is a far more rigourous and complete language than GDScript, offering high level features such as traits or easy packing and unpacking of values. Rust also offers ridiculous performance, so if your game simulation is complex, rust is a good language to run it.

Once you have built your godot-rust project, you explain to Godot where to find the DLL and name each script, each using a little Godot resource file. Godot will open the compiled DLL and execute it when your game runs. See [Getting Started](./getting-started.md) to get an idea how this works.

## Contributing

The source repository for this book is [hosted on GitHub](https://github.com/godot-rust/book).

## License

The `godot-rust` bindings, and this user guide, are licensed under the [MIT license](LICENSE.md).

## Help

There are several active discords which discuss Godot-Rust. [Here is an invite to the official one](https://discord.gg/AbjQp6mjKS). 