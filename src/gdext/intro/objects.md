# Managing objects in Rust

This chapter covers the most central mechanism of the Rust bindings -- one that will accompany you from the Hello-World
example to a sophisticated Rust game.

We're talking about _objects_ and the way they integrate into the Godot engine.


## Table of contents
<!-- toc -->



## Terminology

To avoid confusion, whenever we talk about objects, we mean _instances of Godot classes_. This amounts to `Object` (the hierarchy's root)
and all classes inheriting directly or indirectly from it: `Object`, `Node`, `Resource`, `RefCounted`, etc.

In particular, the term "class" also includes user-provided types that are declared using `#[derive(GodotClass)]`,
even if Rust technically calls them structs. In the same vein, _inheritance_ refers to the conceptual relation
("`Player` inherits `Sprite2D`"), not any technical language implementation.

Objects do **not** include built-in types such as `Vector2`, `Color`, `Transform3D`, `Array`, `Dictionary`, `Variant` etc.
These types, although sometimes called "built-in classes", are not real classes, and we generally do not refer to their instances as _objects_.



### Inheritance

Inheritance is a central concept in Godot. You likely know it already from the node hierarchy, where derived classes add specific functionality.
This concept extends to Rust classes, with inheritance being emulated via composition.

Each Rust class has a base class. You cannot inherit other user-defined classes.

- Typically, a base class is a node type, i.e. it (indirectly) inherits from the class `Node`. This makes it possible to attach instances
  of the class to the scene tree. Nodes are manually managed, so you need to either add them to the scene tree or free them manually.
- If not explicitly specified, the base class is `RefCounted`. This is useful to move data around, without interacting with the scene tree.
  "Data bundles" (collection of multiple fields without much logic) should generally use `RefCounted`.
- `Object` is the root of the inheritance tree. It is rarely used directly, but it is the base class of `Node` and `RefCounted`.
  Use it only when you really need it; it requires manual memory management and is harder to handle.


## The `Gd` smart pointer

[`Gd<T>`][gd] is the type you will encounter the most when working with gdext.

It is also the most powerful and versatile type that the library provides.

In particular, its responsibilities include:
* Holding references to _all_ Godot objects, whether they are engine types like `Node2D` or your own `#[derive(GodotClass)]` structs in Rust.
* Tracking memory management of types that are reference-counted.
* Safe access to user-defined Rust objects through interior mutability.
* Detecting destroyed objects and preventing UB (double-free, dangling pointer, etc.).
* Providing FFI conversions between Rust and engine representations, for engine-provided and user-exposed APIs.

A few practical examples:

1. Retrieve a node relative to current -- type inferred as `Gd<Node3D>`: 
    ```rust
    let child = self.sprite.get_node_as::<Node3D>("Child");
    ```
   
2. Load a scene and instantiate it as a `RigidBody2D`:
    ```rust
    // mob_scene is declared as a field of type Gd<PackedScene>
    self.mob_scene = load("res://Mob.tscn");
    let mut instanced = self.mob_scene.instantiate_as::<RigidBody2D>();
    ```

3. A signal handler for the `body_entered` signal of a `Node3D` in your custom class
    ```rust
    #[godot_api]
    impl Player {
        #[func]
        fn on_body_entered(&mut self, body: Gd<Node3D>) {
            // body holds the reference to the Node3D object that triggered the signal
        }
    }
    ```


## Object lifetime

### Construction

During the [Hello World](hello-world.md) tutorial, we already briefly touched on the `init` function, which can be used to initialize
custom Rust classes.

`init` is called whenever someone requests instantiation of the class. 
This typically happens in two scenarios, let's again assume a class `Player`:

- GDScript code calls `Player.new()`.
- Rust code creates a `Gd<Player>` object.

If `T` is a user-defined type, then a `Gd<T>` object can be created in multiple ways. Check [the `Gd` API docs][gd] for the most up-to-date
constructor list. 

> **Note**: `init` cannot take custom parameters. There are however a few ways around that.
> - From Rust code, simply call [`Gd::with_base()`][gd-with-base].
> - From GDScript, you can write a custom method such as `post_init()`, which late-initializes fields. This will reduce type safety though,
>   needing `Option` for fields whose values aren't available at initialization time.
> - Another alternative is to use a third-party class such as `PlayerFactory`, which has a method accepting parameters and returning a
>   fully-constructed `Player` instance.

    
If your `T` contains a `#[base]` field, you cannot create a standalone `T` object -- you must encapsulate it in `Gd<T>`.
You can also not extract a `T` from a `Gd<T>` smart pointer anymore; since it has potentially been shared with the Godot engine, this would
not be a safe operation.


### Destruction

Once its lifetime ends, the `Gd<T>` object will be dropped, and the destructor (`Drop`) for `T` will be invoked.
Keep in mind that node classes in Godot are manually-managed. When outside the scene tree, they need to be manually deleted
using [`free()`][gd-free].



[gd]: https://godot-rust.github.io/docs/gdext/master/godot/obj/struct.Gd.html
[gd-with-base]: https://godot-rust.github.io/docs/gdext/master/godot/obj/struct.Gd.html#method.with_base
[gd-free]: https://godot-rust.github.io/docs/gdext/master/godot/obj/struct.Gd.html#method.free
