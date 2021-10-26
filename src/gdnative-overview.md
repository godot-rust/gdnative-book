# An Overview of GDNative

GDNative is the interface between the Godot engine and bindings in native languages, such as C, C++ or Rust. It defines how Godot finds and loads dynamic libraries (DLLs / shared objects / dylibs) and understands its contents. 

This chapter gives a broad overview of basic GDNative concepts and godot-rust's approach to implement them in Rust. It is not a usage guide for exposing your Rust code to Godot; see chapter [Binding to Rust code](rust-binding.md) for concrete examples.

When working with GDNative you'll be primarily worried about:
1. `Node` types such as `Node` or `Spatial`  
2. `Resource` types such as `Mesh`, `Texture` or `PackedScene` 
3. Data you can send to or receive from other scripts (perhaps in GDScript) such as `Variant` (eg: values, dictionaries or lists)  
4. How to create your own types, supporting incoming calls.

Subchapters:

1. [Data representations](gdnative-overview/data-representations.md)
1. [`Ref`, `TRef` and `Instance`](gdnative-overview/wrappers.md)