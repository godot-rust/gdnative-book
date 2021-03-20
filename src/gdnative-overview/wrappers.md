# `Ref`, `TRef` and `Instance`

Objects from Godot, such as scene nodes, materials, or other resources are owned and maintained by the Godot engine. This means that your Rust code will store references to existing objects, not values. godot-rust provides special wrapper types to deal with these references, which are explained in this page.

These classes stand in contrast to value types like `bool`, `int`, `Vector2`, `Color` etc., which are copied in GDScript, not referenced. In Rust, those types either map to built-in Rust types or structs implementing the `Copy` trait.

## `Ref`: persistent reference

The generic smart pointer `gdnative::Ref<T, Access>` allows you to store `Object` instances in Rust. It comes with different access policies, depending on how the memory of the underlying object is managed (consult [the docs](https://docs.rs/gdnative/latest/gdnative/struct.Ref.html) for details). Most of the time, you will be working with `Ref<T>`, which is the same as `Ref<T, Shared>` and the only access policy that is explained here. Its memory management mirrors that of the underlying type:
* for all Godot objects inheriting the `Reference` class, `Ref<T>` is reference-counted like `Arc<T>` and will clean up automatically.
* for all other types (i.e. the type `Object` and inheritors of `Node`), `Ref<T>` behaves like a raw pointer with manual memory management.

For example, storing a reference to a Godot `Node2D` instance in a struct would look as follows:
```rust
struct GodotNode {
	node_ref: Ref<Node2D>,
}
```

_See `Ref` in
[godot-rust docs](https://docs.rs/gdnative/latest/gdnative/struct.Ref.html)_


## `TRef`: temporary reference

While `Ref` is a persistent pointer to retain references to Godot objects for an extended period of time, it doesn't grant access to the underlying Godot object. The reason for this is that `Ref` cannot generally guarantee that the underlying object, which is managed by the Godot engine, is valid at the time of the access. However, you as a user are in control of GDScript code and the scene tree, thus you can assert that an object is valid at a certain point in time by using `assume_safe()`. This is an unsafe function that returns a `gdnative::TRef<T, Access>` object, which allows you to call methods on the node. You are responsible for this assumption to be correct; violating it can lead to undefined behavior.

The following example demonstrates `TRef`. A node is stored inside a Rust struct, and its position is modified through `set_position()`. This approach could be used in an ECS (Entity-Component-System) architecture, where `GodotNode` is a component, updated by a system.
```rust
struct GodotNode {
    node_ref: Ref<Node2D>,
}

fn update_position(node: &GodotNode) {
    let pos = Vector2::new(20, 30);
  
    // fetch temporary reference to the node
    let node: TRef<Node2D> = unsafe { node.node_ref.assume_safe() };
    
    // call into the Godot engine
    // this implicitly invokes deref(), turning TRef<Node2D> into &Node2D
    node.set_position(pos);
}
```
Note that the parameter type is `&GodotNode`, not `&mut GodotNode`. Then why is it possible to mutate the Godot object?

All Godot classes in Rust (`Object` and its subtypes) have only methods that operate on `&self`, not `&mut self`. The reason for this choice is that `&mut` is -- strictly speaking -- not a mutable reference, but rather an [_exclusive_ reference](https://docs.rs/dtolnay/latest/dtolnay/macro._02__reference_types.html). The one and only thing it guarantees is that while it exists, no other reference to the same object can exist (no aliasing). Since all the Godot classes can be shared with the Godot engine, which is written in C++ and allows free aliasing, using `&mut` references would potentially violate the exclusivity, leading to UB. This is why `&T` is used, and just like e.g. `&RefCell` it _does_ allow mutation.

This being said, it can still make sense to bring back some type safety on a higher level in your own code. For example, you could make the `update_position()` take a `&mut GodotNode` parameter, to make sure that access to this `GodotNode` object is exclusive.


_See `TRef` in
[godot-rust docs](https://docs.rs/gdnative/latest/gdnative/struct.TRef.html)_


## `Instance`: reference with attached Rust class

When working with classes that are provided by the engine or defined in GDScript, the `Ref` smart pointer is the ideal type for interfacing between Rust and Godot. However, when defining a custom class in Rust, that is registered with the Godot engine, there are two parts that need to be stored together:

1. **GDNative script:** the Rust struct object that implements the entire custom logic. The Rust struct is written by you.
1. **Base object:** the base class from which the script inherits, with its own state. This is always a Godot built-in class such as `Object`, `Reference` or `Node`.

The `Instance` class simply wraps the two parts into a single type.

When passing around your own Rust types, you will thus be working with `Instance`. The traits `ToVariant`, `FromVariant` and `OwnedToVariant` are automatically implemented for `Instance` types, allowing you to pass them from and to the Godot engine.


### Construction

Let's use a straightforward example: a player with name and score. Exported methods and properties are omitted for simplicity; the full interfacing will be explained later in [Calling into GDScript from Rust](../rust-binding/calling-gdscript.md).
```rust
#[derive(NativeClass)]
// no #[inherit], thus inherits Reference by default
pub struct Player {
    name: String,
    score: u32,
}

#[methods]
impl Player {
    fn new(_owner: &Reference) -> Self {
        Self {
            name: "New player".to_string(),
            score: 0
        }
    }
}
```

To create a default instance, use `Instance::new_instance()`.  
You can later use `map()` and `map_mut()` to access the `Instance` immutably and mutably.

```rust
let instance: Instance<Reference, Unique> = Instance::new();
// or:
let instance = Player::new_instance();

// note: map_mut() takes &self, so above is not 'let mut'
instance.map_mut(|p: &mut Player, _base: TRef<Reference, Unique>| {
    p.name = "Joe".to_string();
    p.score = 120;
});
```

If you don't need a Godot-enabled default constructor, use the `#[no_constructor]` attribute and define your own Rust `new()` constructor.
```rust
#[derive(NativeClass)]
#[no_constructor]
pub struct Player {
    name: String,
    score: u32,
}

#[methods]
impl Player {
    pub fn new(name: &str, score: u32) -> Self {
       Self { name: name.to_string(), score }
    }
}
```

In this case, you can construct an `Instance` from an existing Rust object using `Instance::emplace()`:
```rust
let player = Player::new("Joe", 120);

let instance = Instance::emplace(player);
// or:
let instance = player.emplace();
```




_See `Instance` in
[godot-rust docs](https://docs.rs/gdnative/latest/gdnative/nativescript/struct.Instance.html)_

