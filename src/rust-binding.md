# Binding to Rust code

This chapter provides an exhaustive list of mechanisms to pass data through the Rust GDNative binding, in both directions: 
* **GDScript -> Rust**, e.g. to react to an input event with custom Rust logic
* **Rust -> GDScript**, e.g. to apply a game logic change to a graphics node in Godot

The goal is to serve as both an in-depth learning resource for newcomers and a reference to look up specific mechanisms at a later stage. Before delving into this chapter, make sure to read [An Overview of GDNative](gdnative-overview.md), which explains several fundamental concepts used here.

The subchapters are intended to be read in order, but you can navigate to them directly:

1. [Class registration](rust-binding/classes.md)
1. [Exported methods](rust-binding/methods.md)
1. [Exported properties](rust-binding/properties.md)
1. [Calling into GDScript from Rust](./rust-binding/calling-gdscript.md)