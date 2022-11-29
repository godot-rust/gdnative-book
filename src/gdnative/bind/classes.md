# Class registration

Classes are a fundamental data type of GDNative. They are used for Godot's own types (such as nodes) as well as custom ones defined by you. Here, we focus on defining _custom_ classes and exposing them to Godot.


## The Rust entry point

When working with godot-rust, your Rust code sits inside a dynamic library with C ABI (`cdylib`), which is loaded at runtime from the Godot engine. The engine works as the host application with the entry and exit point, and your Rust code will be loaded at some point after Godot starts and unloaded before it ends.

This workflow implies that when you want to execute Rust code, you need to first pass control from Godot to it. To achieve this, every godot-rust application integrated with the engine must expose a public interface, through which Godot can invoke Rust code.

Somewhere in your code, usually in `lib.rs`, you need to declare the functions that will be called by the engine when the native library is loaded and unloaded, as well as the registration function for native classes exposed to the engine. godot-rust provides the following macros (consult [their documentation](https://docs.rs/gdnative/latest/gdnative/index.html#macros) for further info and customization):
```rust
godot_gdnative_init!();
godot_nativescript_init!(init);
godot_gdnative_terminate!();
```
Or the equivalent short-hand:
```rust
godot_init!(init);
```

The argument `init` refers to the function registering native script classes, which is also defined by you. For this chapter, let's assume you want to write a class `GodotApi`, which exposes a public interface to be invoked from Godot. The registration is then as follows:
```rust
// see details later
struct GodotApi { ... }

fn init(handle: InitHandle) {
    handle.add_class::<GodotApi>();
}
```


## Class definition

Similar to the [Hello World](../getting-started/hello-world.md#overriding-a-godot-method) example, we can define the `GodotApi` native class as follows:
```rust
// Tell godot-rust that this struct is exported as a native class 
// (implements NativeClass trait)
#[derive(NativeClass)]

// Specify the base class (corresponds to 'extends' statement in GDScript).
// * Like 'extends' in GDScript, this can be omitted. 
//   In that case, the 'Reference' class is used as a base.
// * Unlike 'extends' however, only existing Godot types are permitted,
//   no other user-defined scripts.
#[inherit(Node)]
pub struct GodotApi {}

// Exactly one impl block can have the #[methods] annotation, 
// which registers methods in the background.
#[methods]
impl GodotApi {
    // Constructor, either:
    fn new(base: &Node) -> Self { ... }
    // or:
    fn new(base: TRef<Node>) -> Self { ... }
}
```

The [`#[derive(NativeClass)]` macro](https://docs.rs/gdnative/latest/gdnative/derive.NativeClass.html) enables a Rust type to be usable as a _native class_ in Godot. It implements [the `NativeClass` trait](https://docs.rs/gdnative/latest/gdnative/nativescript/trait.NativeClass.html), which fills in the glue code required to make the class available in Godot. Among other information, this includes class name and registry of exported methods and properties. For the user, the utility methods `new_instance()` and `emplace()` are provided for constructing `Instance` objects.

The function `new()` corresponds to `_init()` in GDScript. The _base_ is the base object of the script, and must correspond to the class specified in the `#[inherit]` attribute (or `Reference` if the attribute is absent). The parameter can be a shared reference `&T` or a `TRef<T>`.

With a `new()` method, you are able to write `GodotApi.new()` in GDScript. If you don't need this, you can add the `#[no_constructor]` attribute to the struct declaration.

At this point, arguments cannot be passed into the constructor. Consult [this FAQ entry](../faq.md#passing-additional-arguments-to-a-class-constructor) for available workarounds.