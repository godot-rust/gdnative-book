
# Calling GDScript from Rust

To transport information from Rust back to GDScript, you have three options (of which the last can be achieved in different ways):

* Use return values in exported methods
* Use exported properties
* Call a Godot class method from Rust
    * invoke built-in method as part of the API (e.g. `set_position()` above)
    * invoke custom GDScript method, through `call()` and overloads
    * emit a signal, through `emit_signal()`

Which one you need depends on your goals and your architecture. If you see Rust as a deterministic, functional machine in the sense of _input -> processing -> output_, you could stick to only returning data from Rust methods, and never directly calling a Godot method. This can be limiting however, and depending on your use case you end up manually dispatching back to different nodes on the GDScript side.


## Passing custom classes to GDScript

On the previous pages, we explained how to export a class, so it can be instantiated and called from GDScript. This section explains how to construct a class locally in Rust.

Let's define a class `Enemy` which acts as a simple data bundle, i.e. no functionality. We inherit it from `Reference`, such that memory is managed automatically. In addition, we define `_to_string()` and delegate it to the derived `Debug` trait implementation, to make it printable from GDScript.

```rust
#[derive(NativeClass, Debug)]
// no #[inherit], thus inherits Reference by default
#[no_constructor]
pub struct Enemy {
    #[property]
    pos: Vector2,

    #[property]
    health: f32,

    #[property]
    name: String,
}

#[methods]
impl Enemy {
    #[method]
    fn _to_string(&self) -> String {
        format!("{:?}", self) // calls Debug::fmt()
    }
}
```
Godot can only use classes that are registered, so let's do that:
```rust
fn init(handle: InitHandle) {
    // ...
    handle.add_class::<Enemy>();
}
```
Now, it's not possible to directly return `Enemy` instances in exported methods, so this won't work:
```rust
#[method]
fn create_enemy(&mut self) -> Enemy {...}
```
Instead, you can wrap the object in a `Instance<Enemy, Unique>`, using `emplace()`. For an in-depth explanation of the `Instance` class, read [this section](../overview/wrappers.md#instance-reference-with-attached-rust-class).
```rust
#[method]
fn create_enemy(&self) -> Instance<Enemy, Unique> {
    let enemy = Enemy {
        pos: Vector2::new(7.0, 2.0),
        health: 100.0,
        name: "MyEnemy".to_string(),
    };
    
    enemy.emplace()
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

If you want types to be default-constructible, e.g. to allow construction from GDScript, omit the `#[no_constructor]` attribute. You can default-construct from Rust using `Instance::new_instance()`.


## Function calls

While you can export Rust methods to be called from GDScript, the opposite is also possible. Typical use cases for this include:

* Read or modify the scene tree directly from Rust (e.g. moving a node)
* Synchronizing logic state (Rust) with visual representation (Godot)
* Notify code in GDScript about changes in Rust

Methods provided by Godot classes are mapped to regular Rust functions. Examples for these are `Node2D::rotate()`, `Button::set_text()`, `StaticBody::bounce()`. They can usually be invoked safely on a `&T` or `TRef<T>` reference to the respective object.

Custom GDScript methods (defined in .gd files) on the other hand need to be invoked dynamically. This means that there is no type-safe Rust signature, so you will use the `Variant` type. All Godot-compatible types can be converted to variants. For the actual call, `Object` provides multiple methods. Since every Godot class eventually dereferences to `Object`, you can invoke them on any Godot class object.

The following GDScript method:
```python
# in UserInterface.gd
extends CanvasItem

func update_stats(mission_name: String, health: float, score: int) -> bool:
    # ...
```
can be invoked from Rust as follows:
```rust
use gdnative::core_types::ToVariant;

fn update_mission_ui(ui_node: Ref<CanvasItem>) {
    let mission_name = "Thunderstorm".to_variant();
    let health = 37.2.to_variant();
    let score = 140.to_variant();

    // both assume_safe() and call() are unsafe
    let node: TRef<CanvasItem> = unsafe { ui_node.assume_safe() };
    let result: Variant = unsafe {
        node.call("update_stats", &[mission_name, health, score])
    };
  
    let success: bool = result.try_to_bool().expect("returns bool");
}
```

Besides [`Object::call()`](https://docs.rs/gdnative/latest/gdnative/api/struct.Object.html#method.call), alternative methods `callv()` (accepting a `VariantArray`) and `call_deferred()` (calling the next frame) exist, but the principle stays the same.

For long parameter lists, it often makes sense to bundle related functionality into a new class, let's say `Stats` in the above example. When working with classes, you can convert both `Ref` (for Godot/GDScript classes) and `Instance` (for native classes) to `Variant` by means of the `OwnedToVariant` trait:

```rust
use gdnative::core_types::OwnedToVariant; // owned_to_variant()
use gdnative::nativescript::NativeClass; // emplace()

// Native class bundling the update information
#[derive(NativeClass)]
#[no_constructor]
struct Stats {
    #[property]
    mission_name: String,
  
    #[property]
    health: f32,
  
    #[property]
    score: i32,
}

fn update_mission_ui(ui_node: Ref<CanvasItem>) {
    let stats = Stats {
        mission_name: "Thunderstorm".to_variant(),
        health: 37.2.to_variant(),
        score: 140.to_variant(),      
    };
  
    let instance: Instance<Stats, Unique> = stats.emplace();
    let variant: Variant = instance.owned_to_variant();

    let node: TRef<CanvasItem> = unsafe { ui_node.assume_safe() };
    
    // let's say the method now returns a Stats object with previous stats
    let result: Variant = unsafe { node.call("update_stats", &[variant]) };
  
    // convert Variant -> Ref -> Instance
    let base_obj: Ref<Reference> = result.try_to_object().expect("is Reference");
    let instance: Instance<Stats, Shared> = Instance::from_base(base_obj).unwrap();
    
    instance.map(|prev_stats: &Stats, _base| {
        // read prev_stats here
    });
}
```

### Warning

When calling GDScript functions from Rust, a few things need to be kept in mind.

**Safety:** Since the calls are dynamic, it is possible to invoke any other functions through them, including unsafe ones like `free()`. As a result, `call()` and its alternatives are unsafe.

**Re-entrancy:** When calling from Rust to GDScript, your Rust code is usually already running in an exported `#[method]` method, meaning that it has bound its receiver object via `&T` or `&mut T` reference. In the GDScript code, you must not invoke any method on the same Rust receiver, which would violate safety rules (aliasing of `&mut`).


## Signal emissions

Like methods, signals defined in GDScript can be emitted dynamically from Rust.

The mechanism works analogously to function invocation, except that you use [`Object::emit_signal()`](https://docs.rs/gdnative/latest/gdnative/api/struct.Object.html#method.emit_signal) instead of `Object::call()`.