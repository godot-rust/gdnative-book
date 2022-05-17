# FAQ: Godot 4.0 Status

## Table of contents
<!-- toc -->

## What is the status of Godot 4 Support?

Currently we are still in the planning phase of determining how to support Godot 4.0 and where we will focus our efforts in the future.  
You can follow [this GitHub issue](https://github.com/godot-rust/godot-rust/issues/824) for updates.



## What is GDExtension? Why aren't we just upgrading GDNative?

The Godot team has officially announced [GDExtension](https://godotengine.org/article/introducing-gd-extensions) as the **replacement** for GDNative. GDNative is no longer supported in Godot 4.


## What do we know so far?

Currently as there is limited documentation about how to implement [GDExtension](https://godotengine.org/article/introducing-gd-extensions), most of what we know is being explored via the [Godot C++ bindings repo](https://github.com/godotengine/godot-cpp) and determining how to do the following:

- Generate the language bindings
- How do we register classes with GDExtension along with the rest of the data?
- How to migrate to the extensions.
- Determine how this needs to integrate with Rust.
- Which core types need to be reimplemented.

In addition, we still need to start investigating and planning how the procedural macros need to be changed or rewritten to support a similar level of ergonomics that `gdnative` currently offers.


## How can I help?

We would be grateful for any help, please feel free to reach out to us on any of our [community platforms](https://godot-rust.github.io/community/).