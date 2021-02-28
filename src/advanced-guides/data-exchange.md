# Exchanging data between GDScript and Rust

This chapter aims at providing an exhaustive list of mechanisms to pass data between Rust and GDScript. To understand the underlying terminology, make sure to read the chapter [Data representations](data-representations.md) first.

* [Passing data from GDScript to Rust](#passing-data-from-gdscript-to-rust)
  * [The Rust entry point](#the-rust-entry-point)
  * [Exported classes](#exported-classes)
  * [Exported methods](#exported-methods)
  * [Exported properties](#exported-properties)
* [Passing data from Rust to GDScript](#passing-data-from-rust-to-gdscript)  
  * [Class construction in Rust](#class-construction-in-rust)
  
_For readability, code blocks use the `gd` module name in place of `gdnative`. Its submodules are listed with full name, for clarity._  
_In your own code, you may consider to use the prelude:_
```rust
mod gd {
    pub use gdnative::prelude::*;
}
```


# Passing data from GDScript to Rust

When working with godot-rust, your Rust code sits inside a dynamic library with C ABI (`cdylib`), which is loaded at runtime from the Godot engine. The engine works as the host application with the entry and exit point, and your Rust code will be loaded at some point after Godot starts and unloaded before it ends.

This workflow implies that when you want to execute Rust code, you need to first pass control from Godot to it. To achieve this, every godot-rust application integrated with the engine must expose a public interface, through which Godot can invoke Rust code. 


## The Rust entry point

Somewhere in your code, usually in `lib.rs`, you need to declare the functions that will be called by the engine when the native library is loaded and unloaded, as well as the registration function for native classes exposed to the engine. godot-rust provides the following macros (consult [their documentation](https://docs.rs/gdnative/latest/gdnative/index.html#macros) for further info and customization):
```rust
gd::godot_gdnative_init!();
gd::godot_nativescript_init!(init);
gd::godot_gdnative_terminate!();
```

The argument `init` refers to the function registering native script classes, which is also defined by you. Let's assume you have one class `GodotApi`, which exposes a public interface to be invoked from Godot. The registration is then as follows:
```rust
fn init(handle: gd::nativescript::InitHandle) {
	handle.add_class::<GodotApi>();
}
```

## Exported classes

Let's look again at a simple example, very similar to the [Hello World](../getting-started/hello-world.html#overriding-a-godot-method) one:
```rust
// Tell godot-rust that this struct is exported as a native class 
// (implements NativeClass trait)
#[derive(gd::nativescript::NativeClass)]

// Specify the base class (corresponds to 'extends' statement in GDScript).
// Like 'extends' in GDScript, this can be omitted. In that case, the 
// 'Reference' class is used as a base.
// Unlike 'extends' however, only existing Godot types are permitted,
// no other user-defined scripts.
#[inherit(gd::api::Node)]
pub struct GodotApi {}

// One impl block can have the #[methods] annotation, 
// which registers methods in the background 
#[gd::methods]
impl GodotApi {
    // Constructor, either:
    fn new(owner: &gd::api::Node) -> Self { ... }
    // or:
    fn new(owner: gd::TRef<gd::api::Node>) -> Self { ... }
}
```

The function `new()` corresponds to `_init()` in GDScript. The _owner_ is the base object of the script, and must correspond to the class specified in the `#[inherit]` attribute (or `gd::api::Reference` if the attribute is absent). The parameter can be a shared reference `&T` or a `TRef<T>`. 

With a `new()` method, you are able to write `GodotApi.new()` in GDScript. If you don't need this, you can add the `#[no_constructor]` attribute to the struct declaration.

At this point, arguments cannot be passed into the constructor. There are two main workarounds:
* Declare an `initialize()` method or similar -- this however requires having uninitialized state for a short time, which may decrease type safety (e.g. if you need to use `Option` for a field, before assigning a value)
* Register a separate class, which returns fully-constructed instances. So instead of `Enemy.new()` you can write `EnemyFactory.create(args)`.


## Exported methods

In order to pass data from Godot to Rust, you can export methods. With the `#[export]` attribute, godot-rust takes care of method registration and serialization. Note that the constructor is not annotated with `#[export]`.

```rust
#[derive(gd::nativescript::NativeClass)]
#[inherit(gd::api::Node)]
pub struct GodotApi {
    enemy_count: i32,
}

#[gd::methods]
impl GodotApi {
    fn new(_owner: &gd::api::Node) -> Self {
        // Print to both shell and Godot editor console
        gd::godot_print!("_init()");
        Self { enemy_count: 0 }
    }
    
    #[gd::export]
    fn create_enemy(
        &mut self,
        _owner: &gd::api::Node,
        typ: String,
        pos: gd::core_types::Vector2
    ) {
        gd::godot_print!("create_enemy(): type '{}' at position {:?}", typ, pos);
        self.enemy_count += 1;
    }

    #[gd::export]
    fn create_enemy2(
        &mut self,
        _owner: &gd::api::Node,
        typ: gd::core_types::GodotString,
        pos: gd::core_types::Variant
    ) {
        gd::godot_print!("create_enemy2(): type '{}' at position {:?}", typ, pos);
        self.enemy_count += 1;
    }

    #[gd::export]
    fn count_enemies(&self, _owner: &gd::api::Node) -> i32 {
        self.enemy_count
    }  
}
```
These two methods are semantically equivalent, yet they demonstrate how godot-rust implicitly converts the values to the parameter types (unboxing). You could use `Variant` everywhere, however it is more type-safe and expressive to use specific types. The same applies to return types, you could use `Variant` instead of `i32`. 
`GodotString` is the Godot engine string type, but it can be converted to standard `String`. To choose between the two, consult [the docs](https://docs.rs/gdnative/latest/gdnative/core_types/struct.GodotString.html).  

In GDNative, you can then write this code:
```python
var api = GodotApi.new()

api.create_enemy("Orc", Vector2(10, 20));
api.create_enemy2("Elf", Vector2(50, 70));

print("enemies: ", api.count_enemies())

# don't forget to add it to the scene tree, otherwise memory must be managed manually 
self.add_child(api)
```

The output is:
```
_init()
create_enemy(): type 'Orc' at position (10.0, 20.0)
create_enemy2(): type 'Elf' at position Vector2((50, 70))
enemies: 2
```

If you need to override a Godot lifecycle method, just declare it as a normal exported method, with the same name and signature as in GDScript:
```rust
#[gd::export]
fn _ready(&mut self, _owner: &gd::Node) {...}

#[gd::export]
fn _process(&mut self, _owner: &gd::Node, delta: f32) {...}

#[gd::export]
fn _physics_process(&mut self, _owner: &gd::Node, delta: f32) {...}

#[gd::export] // extremely useful
fn _to_string(&self, _owner: &gd::Reference) -> String {...}
```


## Exported properties

Like methods, properties can be exported. The `#[property]` attribute above a field declaration makes the field available to Godot, with its name and type.

In the previous example, we could replace the `count_enemies()` method with a property `enemy_count`.
```rust
#[derive(gd::nativescript::NativeClass)]
#[inherit(gd::api::Node)]
pub struct GodotApi {
    #[property]
    enemy_count: i32,
}
```

The GDScript code would be changed as follows.
```python
print("enemies: ", api.enemy_count)
```

That's it.


## `Ref` and `TRef` smart pointers

The above examples have dealt with simple types such as strings and integers. What if we want to pass entire classes to Rust?

The generic type `Ref` allows you to store `Object` instances in Rust. It comes with different access policies, depending on how the memory of the underlying object is managed (consult [the docs](https://docs.rs/gdnative/latest/gdnative/struct.Ref.html) for details). Most of the time, you will be working with `Ref<T>`, which is the same as `Ref<T, Shared>` and the only access policy that is explained here. Its memory management mirrors that of the underlying type:
* for `T: Reference` (i.e. all Godot objects inheriting the `Reference` class), `Ref<T>` is reference-counted like `Arc<T>` and will clean up automatically.
* for all other types (i.e. `T = Object` and `T: Node`) `Ref<T>` behaves like a raw pointer with manual memory management.

For example, if we want to pass in an enemy from GDScript instead of creating one locally, we could do the following. Let's assume an enemy is represented by the `Node2D` class; the same principle applies to custom classes.
```rust
#[derive(gd::nativescript::NativeClass)]
#[inherit(gd::api::Node)]
pub struct GodotApi {
    // Store references to all enemy nodes
    enemies: Vec<gd::Ref<gd::api::Node2D>>,
}

#[gd::methods]
impl GodotApi {
    // ...

    #[gd::export]
    fn add_enemy(
        &mut self,
        _owner: &gd::Node,
        enemy: gd::Ref<gd::api::Node2D> // pass in enemy
    ) {
       self.enemies.push(enemy);
    }
  
    // You can even return the enemies directly with Vec
    // This yields an array of nodes on the GDScript side
    // Alternative in the general case: VariantArray
    #[gd::export]
    fn get_enemies(
      &self,
      _owner: &gd::api::Node
    ) ->  Vec<gd::Ref<gd::api::Node2D>> {
      self.enemies.clone()
    }
}
```

While `Ref` is a persistent pointer allowing you to retain references to Godot objects for an extended period of time, it doesn't grant access the underlying Godot object. The reason for this is that `Ref` cannot generally guarantee that the underlying object, which is managed by the Godot engine, is valid at the time of the access. However, you as a user are in control of GDScript code and the scene tree, thus you can assert that an object is valid at a certain point in time by using `assume_safe()`. This is an unsafe function that returns a `TRef<T>` object, which allows you to call methods on the node.

The following example demonstrates `TRef`. A node is stored inside a Rust struct, and its position is modified through `set_position()`. This approach could be used in an ECS, where `GodotNode` is a component, updated by a system.
```rust
struct GodotNode {
    node_ref: gd::Ref<gd::api::Node2D>,
}

fn update_position(node: &GodotNode) {
    let pos = gd::core_types::Vector2::new(20, 30);
  
    // fetch temporary reference to the node
    let node: gd::TRef<gd::api::Node2D> = unsafe { node.node_ref.assume_safe() };
    
    // call into the Godot engine
    // this implicitly invokes deref(), turning TRef<Node2D> into &Node2D
    node.set_position(pos);
}
```
Note that the parameter type is `&GodotNode`, not `&mut GodotNode`. Then why is it possible to mutate the Godot object?

All Godot classes in Rust (`Object` and its subtypes) have only methods that operate on `&self`, not `&mut self`. The reason for this choice is that `&mut` is strictly not a mutable reference, but rather an [_exclusive_ reference](https://docs.rs/dtolnay/latest/dtolnay/macro._02__reference_types.html). The one and only thing it guarantees is that while it exists, no other reference to the same object can exist (no aliasing). Since all the Godot classes can be shared with the Godot engine, which is written in C++ and allows free aliasing, using `&mut` references would potentially violate the exclusivity, leading to UB. This is why `&T` is used, and just like `&RefCell` it _does_ allow mutation.

This being said, it can still make sense to bring back some type safety on a higher level in your own code. For example, you could make the `update_position()` take a `&mut GodotNode` parameter, to make sure that access to this `GodotNode` object is exclusive.

_See `Ref` in
[godot-rust docs](https://docs.rs/gdnative/latest/gdnative/struct.Ref.html)_  
_See `TRef` in
[godot-rust docs](https://docs.rs/gdnative/latest/gdnative/struct.TRef.html)_

----


# Passing data from Rust to GDScript

To transport information from Rust back to GDScript, you have three options (of which the last can be achieved in different ways):

* Use return values in exported methods
* Use exported properties
* Call a Godot class method from Rust
  * invoke built-in method as part of the API (e.g. `set_position()` above)
  * invoke custom GDScript method, through `call()` and overloads
  * emit a signal, through `emit_signal()`

Which one you need depends on your goals and your architecture. If you see Rust as a deterministic, functional machine in the sense of _input -> processing -> output_, you could stick to only returning data from Rust methods, and never directly calling a Godot method. This can be limiting however, and depending on your use case you end up manually dispatching back to different nodes on the GDScript side.


## Class construction in Rust

Above, we explained how to export a class, so it can be instantiated and called from GDScript. This section explains how to construct a class locally in Rust.

Let's define a class `Enemy` which acts as a simple data bundle, i.e. no functionality. We inherit it from `Reference`, such that memory is managed automatically. In addition, we define `_to_string()` and delegate it to the derived `Debug` trait implementation. 

```rust
#[derive(gd::NativeClass, Debug)]
#[inherit(gd::api::Reference)]
#[no_constructor]
pub struct Enemy {
	#[property]
	pos: gd::core_types::Vector2,

	#[property]
	health: f32,

	#[property]
	name: String,
}

#[gd::methods]
impl Enemy {
	#[gd::export]
	fn _to_string(&self, _owner: &gd::Reference) -> String {
		format!("{:?}", self)
	}
}
```
Godot can only use classes that are registered, so let's do that:
```rust
fn init(handle: gd::nativescript::InitHandle) {
    // ...
    handle.add_class::<Enemy>();
}
```
Now, it's not possible to directly return `Enemy` instances in exported methods, so this won't work:
```rust
#[gdnative::export]
fn create_enemy(&mut self, _owner: &gd::Node) -> Enemy {...}
```
One way would be to implement the `ToVariant` trait, but much easier is to wrap the object in a `Instance<Enemy, Unique>`.  
This can easily be done with `emplace()`:
```rust
#[gdnative::export]
fn create_enemy(
    &self,
    _owner: &gd::Node
) -> gd::Instance<Enemy, gd::thread_access::Unique> {
    let enemy = Enemy {
        pos: gd::Vector2::new(7.0, 2.0),
        health: 100.0,
        name: "MyEnemy".to_string(),
    };
    
    gd::NativeClass::emplace(enemy)
    // or, with 'use gd::nativescript::NativeClass;'
    Enemy::emplace(enemy)
}
```

When calling this method in GDScript:
```python
var api = GodotApi.new()
var enemy = api.create_enemy()
print("Enemy created: ", enemy)
```
the output will be:
``` 
Enemy created: Enemy { pos: (7.0, 2.0), health: 100.0, name: "MyEnemy" }
```

If you need default-constructible classes (e.g. to allow construction from GDScript), omit the `#[no_constructor]` attribute and define a canonical `new()` constructor as usual. It can often make sense to combine this with the `Default` trait:
```rust
#[derive(gd::NativeClass, Debug, Default)]
#[inherit(gd::api::Reference)]
pub struct Enemy {...}

#[gd::methods]
impl Enemy {
	fn new(_owner: &gd::Reference) -> Self {
		Default::default()
	}
}
```

In this case, you can construct a default instance with `new_instance` instead of `emplace()`. Modification becomes a bit more verbose, using `map_mut()`:
```rust
#[gdnative::export]
fn create_enemy(
    &self,
    _owner: &gd::Node
) -> gd::Instance<Enemy, gd::thread_access::Unique> {
    let instance = Enemy::new_instance();

    instance.map_mut(|
        enemy: &mut Enemy,
        _owner: gd::TRef<gd::api::Reference, gd::thread_access::Unique>
    | {
        enemy.pos = gd::Vector2::new(7.0, 2.0);
        enemy.health = 100.0;
        enemy.name = "MyEnemy".to_string();
    });

    instance
}
```