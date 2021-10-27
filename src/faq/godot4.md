# FAQ: Godot 4.0 Status

## What is the status of Godot 4 Support?

Currently we are still in the planning phase of determining how to support Godot 4.0 and where we will focus our efforts in the future.


## So what is the plan?

We don't have any specifics at this time. This page will be updated as soon as we have a more concrete plan to share.


## What is GDExtension? Why aren't we just upgrading GDNative?

Currently the Godot team has officially announced [GDExtension](https://godotengine.org/article/introducing-gd-extensions) as the __replacement__ for GDNative. GDNative appears to be no longer supported in Godot 4.0.


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