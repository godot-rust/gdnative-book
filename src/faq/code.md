# FAQ: Common code questions

## Table of contents
<!-- toc -->

## How do I store a reference of `Node`?

The idiomatic way to maintain a reference to a node in the SceneTree from Rust is to use `Option<Ref<T>>`.

For example, the following GDScript code:
```gdscript
extends Node
class_name MyClass
var node

func _ready():
  node = Node.new()
  self.add_child(node, false)

```

could be translated to this Rust snippet:
```rust
#[derive(NativeClass)]
#[inherit(Node)]
#[no_constructor]
struct MyNode {
    node_ref: Option<Ref<Node>>
}

#[methods]
impl MyNode {
    #[export]
    fn _ready(&self, owner: TRef<Node>) {
        let node = Node::new();
        owner.add_child(node);
        self.node_ref = Some(node.claim());
    }
}
```

Note: As `TRef<T>` is a temporary pointer, it will be necessary to get the base pointer `Ref<T>` in order to continue to hold this value.

This can be done with the [`TRef<T>::claim()`](https://docs.rs/gdnative/latest/gdnative/struct.TRef.html#method.claim) function that returns the persistent version of the pointer, which you can store in your class.


## Borrow failed; a &mut reference was requested

In Rust, [there can only be *one* `&mut` reference to the same memory location at the same time](https://docs.rs/dtolnay/0.0.9/dtolnay/macro._02__reference_types.html). To enforce this while making simple use cases easier, the bindings make use of [interior mutability](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html). This works like a lock: whenever a method with `&mut self` is called, it will try to obtain a lock on the `self` value, and hold it *until it returns*. As a result, if another method that takes `&mut self` is called in the meantime for whatever reason (e.g. signals), the lock will fail and an error (`BorrowFailed`) will be produced.

It's relatively easy to work around this problem, though: Because of how the user-data container works, it can only see the outermost layer of your script type - the entire structure. This is why it's stricter than what is actually required. If you run into this problem, you can [introduce finer-grained interior mutability](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html) in your own type, and modify the problematic exported methods to take `&self` instead of `&mut self`.

This issue also can often occur when using signals to indicate an update such as in the following code.

```rust
#[derive(NativeClass)]
#[inherit(Node)]
// register_with attribute can be used to specify custom register function for node signals and properties
#[register_with(Self::register_signals)]
struct SignalEmitter {
    data: LargeData,
}

#[methods]
impl SignalEmitter {
    fn register_signals(builder: &ClassBuilder<Self>) {
        builder.add_signal(Signal { name: "updated", args: &[] });
    }

    fn new(_owner: &Node) -> Self {
        SignalEmitter {
            data: "initial",
        }
    }
    #[export]
    fn update_data(&mut self, owner: TRef<Node>, data: LargeData) {
        self.data = data;
        owner.emit_signal("updated", &[]);
    }
    #[export]
    fn get_data(&self, owner: TRef<Node>) -> &LargeData {
        &self.data
    }
}
```

The assumption with the above code is that `SignalEmitter` is holding data that is too large to be feasible to clone into the signal. So the purpose of the signal is to notify other Nodes or Objects that this the data has been updated.

The problem is that, unless the nodes all connect with the `Object::CONNECT_DEFERRED` flag, they will be notified immediately and will attempt to borrow the data. This is the root cause of the `BorrowFailed` error.

There are two ways to solve it.

1. Ensure that all nodes use `Object::CONNECT_DEFERRED` since this will ensure that the callbacks wait until the idle_frame signal to borrow the data.
2. Store `data` in a `RefCell<LargeData>` if it should only be accessed from the same thread (such as with signals) or `Mutex<LargeData>` if you need thread-safety. Then you can modify `update_data()` to the following snippet:

```rust
    fn update_data(&self, owner: TRef<Node>, data: LargeData) {
        // If using RefCell
        self.data.replace(data);
        // If using Mutex
        // *self.data.lock().expect("this should work") = data;
        owner.emit_signal("updated", &[]);
    }
```

In both instances you will not encounter the reentrant errors.


## Why do mutating Godot methods take `&self` and not `&mut self`?

1. `&mut` means that only one reference can exist simultaneously (no aliasing), _not_ that the object is mutable. Mutability is often a _consequence_ of the exclusiveness. Since Godot objects are managed by the engine, Rust cannot guarantee that references are exclusive, as such using `&mut` would cause undefined behavior. Instead, an interior mutability pattern based on `&T` is used. For godot-rust, it is probably more useful to consider `&` and `&mut` as "shared" and "unique" respectively.
  For more information, please refer to [this explanation](https://docs.rs/dtolnay/latest/dtolnay/macro._02__reference_types.html) for more information.
2. Why godot-rust does not use `RefCell` (or some other form of interior mutability) is because the types already have interior mutability as they exist in Godot (and can't be tracked by Rust). For example, does `call()` modify its own object? This depends on the arguments. There are [many such cases](https://github.com/godot-rust/godot-rust/issues/808#issuecomment-964034258) which are much more subtle.


## Why is there so much `unsafe` in godot-rust?

Short Answer: Godot is written in C++, which cannot be statically analyzed by the Rust compiler to guarantee safety.

Longer Answer: `unsafe` is required for different reasons:

- **Object lifetimes:** Godot manages memory of objects independently of Rust. This means that Rust references can be invalidated when Godot destroys referred-to objects. This usually happens due to bugs in GDScript code, like calling `free()` of something actively in use.
- **Thread safety:** while Rust has a type system to ensure thread safety statically (`Send`, `Sync`), such mechanisms do not exist in either GDScript or C++. Even user-defined GDScript code has direct access to the `Thread` API.
- **C FFI:** any interactions that cross the C Foreign Function Interface will be unsafe by default as Rust cannot inspect the other side. While many functions may be safely reasoned about, there are still some functions which will be inherently unsafe due to their potential effects on object lifetimes.

One of the ways that godot-rust avoids large `unsafe` blocks is by using the [TypeState pattern](http://cliffle.com/blog/rust-typestate/) with _temporary references_ such as `TRef` and `TInstance`. For more information see [`Ref`, `TRef` and `Instance`](../../src/gdnative-overview/wrappers.md).

Here is an example of some common `unsafe` usage that you will often see and use in your own games.

```rust
fn get_a_node(&self, owner: TRef<Node>) {
    // This is safe because it returns an option that Rust knows how to check.
    let child = owner.get_child("foo");
    // This is safe because Rust panics if the returned `Option` is None.
    let child = child.expect("I know this should exist");
    // This is also safe because Rust panics if the returned `Option` is None.
    let child = child.cast_instance::<Foo>().expect("I know that it must be this type");
    // This is unsafe because the compiler cannot reason about the lifetime of `child`.
    // It is the programmer's responsibility to ensure that `child` is not freed before
    // it gets used.
    let child: Instance<Foo> = unsafe { child.assume_safe() };
    // This is safe because we have already asserted above that we are assuming that
    // there should be no problem and Rust can statically analyze the safety of the
    // functions.
    child.map_mut(|c, o| {
        c.bar(o);
    }).expect("this should not fail");
    // This is unsafe because it relies on Godot for function dispatch and it is
    // possible for it to call `Object.free()` or `Reference.unreference()` as
    // well as other native code that may cause undefined behavior.
    unsafe {
        child.call("bar", &[])
    };
}
```

By the way, safety rules are subject to an [ongoing discussion](https://github.com/godot-rust/godot-rust/issues/808) and likely to be relaxed in future godot-rust versions.


## Can the `new` constructor have additional parameters?

Unfortunately this is currently not possible, due to a general limitation of GDNative (see [related issue](https://github.com/godotengine/godot/issues/23260)).

As a result, a common pattern to work around this limitation is to use explicit initialization methods. For instance:

```rust
struct EnemyData {
    name: String,
    health: f32,
}

#[derive(NativeClass)]
#[inherit(Object)]
struct Enemy {
    data: Option<EnemyData>,
}

#[methods]
impl Enemy {
    fn new(_owner: &Object) -> Self {
        Enemy {
            data: None,
        }
    }

    #[export]
    fn set_data(&mut self, _owner: &Object, name: String, health: f32) {
        self.data = Some(EnemyData { name, health });
    }
}
```

This however has two disadvantages:
1. You need to use an `Option` with the sole purpose of late initialization, and subsequent `unwrap()` calls or checks -- weaker invariants in short.
1. An additional type `EnemyData` for each native class like `Enemy` is required (unless you have very few properties, or decide to add `Option` for each of them, which has its own disadvantages).

An alternative is to register a separate factory class, which returns fully-constructed instances:
```rust
#[derive(NativeClass)]
#[no_constructor] // disallow default constructor
#[inherit(Object)]
struct Enemy {
    name: String,
    health: f32,
}

#[methods]
impl Enemy {
    // nothing here
}

#[derive(NativeClass)]
#[inherit(Reference)]
struct EntityFactory {}

#[methods]
impl EntityFactory {
    #[export]
    fn enemy(&self, _owner: &Object, name: String, health: f32)
    -> Instance<Enemy, Unique> {
        Enemy { name, health }.emplace()
    }
}
```
So instead of `Enemy.new()` you can write `EntityFactory.enemy(args)` in GDScript.
This still needs an extra type `EntityFactory`, however you could reuse that for multiple classes.


## Can I implement static methods in GDNative?

In GDScript, classes can have static methods. However, GDNative currently doesn't allow to register static methods from bindings.

As a work-around, it is possible to use a ZST (zero-sized type):

```rust
#[derive(NativeClass, Copy, Clone, Default)]
#[user_data(Aether<StaticUtil>)]
#[inherit(Object)]
pub struct StaticUtil;

#[methods]
impl StaticUtil {
    #[export]
    fn compute_something(&self, _owner: &Object, input: i32) -> i32 {
        godot_print!("pseudo-static computation");
        2 * input
    }
}
```

[`Aether`](https://docs.rs/gdnative/latest/gdnative/export/user_data/struct.Aether.html) is a special user-data wrapper intended for zero-sized types, that does not perform any allocation or synchronization at runtime.

The type needs to be instantiated somewhere on GDScript level.
Good places for instantiation are for instance:

* a member of a long-living util object,
* a [singleton auto-load object](https://docs.godotengine.org/en/stable/getting_started/step_by_step/singletons_autoload.html).


## How do I convert from a `Variant` to the underlying Rust type?

Assuming that a method takes an argument `my_node` as a `Variant`

You can convert `my_node` to a `Ref`, and then to an `Instance` or `TInstance`, and mapping over it to access the Rust data type:

```rust
/// My class that has data
#[derive(NativeClass)]
#[inherit(Node2D)] // something specific, so it's clear when this type re-occurs in code
struct MyNode2D { ... }

/// Utility script that uses MyNode2D
#[derive(NativeClass, Copy, Clone, Default)]
#[user_data(Aether<AnotherNativeScript>)] // ZST, see above
#[inherit(Reference)]
pub struct AnotherNativeScript;

#[methods]
impl AnotherNativeScript {
    #[export]
    pub fn method_accepting_my_node(&self, _owner: &Reference, my_node: Variant) {
        // 1. Cast Variant to Ref of associated Godot type, and convert to TRef.
        let my_node = unsafe {
            my_node
                .try_to_object::<Node2D>()
                .expect("Failed to convert my_node variant to object")
                .assume_safe()
        };
        
        // 2. Obtain a TInstance which gives access to the Rust object's data.
        let my_node = my_node
            .cast_instance::<MyNode2D>()
            .expect("Failed to cast my_node object to instance");
        
        // 3. Map over the RefInstance to process the underlying user data.
        my_node
            .map(|my_node, _owner| {
                // Now my_node is of type MyNode2D.
            })
            .expect("Failed to map over my_node instance");
    }

}

```

## Is it possible to set subproperties of a Godot type, such as a `Material`?

Yes, it is possible, but it will depend on the type.

For example, when you need to set an `albedo_texture` on the [`SpatialMaterial`](https://docs.rs/gdnative/latest/gdnative/api/struct.SpatialMaterial.html), it will be necessary to use the generic [`set_texture()`](https://docs.rs/gdnative/latest/gdnative/api/struct.SpatialMaterial.html#method.set_texture) function with the parameter `index`.

While a similar case applies for [`ShaderMaterial`](https://docs.rs/gdnative/latest/gdnative/api/struct.ShaderMaterial.html) in the case of shader material, to set the shader parameters, you will need to use the [`set_param()`](https://docs.rs/gdnative/latest/gdnative/api/struct.ShaderMaterial.html#method.set_shader_param) method with the relevant parameter name and value.

Direct access to such properties is planned in [godot-rust/godot-rust#689](https://github.com/godot-rust/godot-rust/issues/689).


## What is the Rust equivalent of `preload`?

Unfortunately, there is no equivalent to preload in languages other than GDScript, because preload is GDScript-specific magic that works at compile time. If you read the official documentation on preload, it says:

>    "Returns a resource from the filesystem that is _loaded during script parsing._" **(emphasis mine)**

This is only possible in GDScript because the parser is deeply integrated into the engine.

You can use [`ResourcePreloader`](https://docs.rs/gdnative/latest/gdnative/api/struct.ResourcePreloader.html) as a separate node in your scene, which will work regardless of whether you use Rust or GDScript. However, note that if you create a `ResourcePreloader` in your code, you will still be loading these resources at the time of execution, because there is no way for the engine to know what resources are being added before actually running the code.

The [`ResourceLoader`](https://docs.rs/gdnative/latest/gdnative/api/struct.ResourceLoader.html) should be used in most cases.

Also, you can go with a `static Mutex<HashMap<..>>` variable and load everything you need there during a loading screen.


## How can function parameters accept Godot subclasses (polymorphism)?

_Static_ (compile-time) polymorphism can be achieved by a combination of the `SubClass` trait and `upcast()`.

For example, let's assume we want to implement a helper function that should accept any kind of [`Container`](https://docs.godotengine.org/en/stable/classes/class_control.html#class-control). The helper function can make use of `SubClass` and `upcast()` as follows:

```rust
fn do_something_with_container<T>(container: TRef<'_, T>)
    where T: GodotObject + SubClass<Container>
// this means: accept any TRef<T> where T inherits `Container`
{
    // First upcast to a true container:
    let container = container.upcast::<Container>();
    // Now you can call `Container` specific methods like:
    container.set_size(...);
}
```

This function can now be used with arbitrary subclasses, for instance:

```rust
fn some_usage() {
    let panel: Ref<PanelContainer> = PanelContainer::new().into_shared();
    let panel: TRef<PanelContainer> = unsafe { panel.assume_safe() };
    do_something_with_container(panel);
}
```

Note that `SubClass` is only a marker trait that models the inheritance relationship of Godot classes, and doesn't perform any conversion by itself. For instance, `x: Ref<T>` or `x: TRef<'_, T>` satisfying `T: GodotObject + SubClass<Container>` doesn't mean that `x` can be used as a `Container` directly. Rather, it ensures that e.g. `x.upcast::<Container>()` is guaranteed to work, because `T` is a subclass of `Container`. Therefore, it is a common pattern to use `SubClass` constraints in combination with `.upcast()` to convert to the base class, and actually use `x` as such.

Of course, you could also delegate the work to upcast to the call site:

```rust
fn do_something_with_container(container: TRef<Container>) { ... }

fn some_usage() {
    let panel: TRef<PanelContainer> = ...;
    do_something_with_container(panel.upcast());
}
```
This would also support _dynamic_ (runtime) polymorphism -- `Ref<T>` can also store subclasses of `T`.


## What is the Rust equivalent of `onready var`?

Rust does not have a direct equivalent to `onready var`. The most idiomatic workaround with Rust is to use `Option<Ref<T>>` of you need the Godot node type or `Option<Instance<T>>` if you are using a Rust based `NativeClass`.

```gdscript
extends Node
class_name MyClass
onready var node = $Node2d
```
You would need to use the following code.

```rust
#[derive(NativeClass, Default)]
#[inherit(Node)]
#[no_constructor]
struct MyNode {
    // late-initialization is modeled with Option
    // the Default derive will initialize both to None 
    node2d: Option<Ref<Node>>,
    instance: Option<Ref<MyClass>>,
}

#[methods]
impl MyNode {
    #[export]
    fn _ready(&self, owner: TRef<Node>) {
        // Get an existing child node that is a Godot class.
        let node2d = owner
            .get_node("Node2D")
            .expect("this node must have a child with the path `Node2D`");
        let node2d = unsafe { node2d.assume_safe() };
        let node2d = node2d.cast::<Node2D>();
        self.node2d = Some(node2d.claim());
      
        // Get an existing child node that is a Rust class.
        let instance = owner
            .get_node("MyClass")
            .expect("this node must have a child with the path `MyClass`");
        let instance = unsafe { instance.assume_safe() };
        let instance = instance.cast_instance::<MyClass>()
                        .expect("child must be type `MyClass`");
        self.instance = Some(instance.claim());
    }
}
```

## What types are supported for passing through the GDNative API?

The GDNative API supports any type that implements the [`ToVariant`](https://docs.rs/gdnative/latest/gdnative/core_types/trait.ToVariant.html) and/or [`FromVariant`](https://docs.rs/gdnative/latest/gdnative/core_types/trait.FromVariant.html) traits.

To use a type as a property, in addition to the above, the type will also need to implement the [`Export`](https://docs.rs/gdnative/latest/gdnative/nativescript/trait.Export.html) trait.

Some concrete examples of types that can be used with the GDNative API are the following:

- [`Variant`](https://docs.rs/gdnative/latest/gdnative/core_types/struct.Variant.html), this is Godot's "any" type. It must be converted before it can be used.
- A subset of scalar types such as `i64`, `f64`, `bool`, etc.
- [`String`](https://doc.rust-lang.org/std/string/struct.String.html) and [`GodotString`](https://docs.rs/gdnative/latest/gdnative/core_types/struct.GodotString.html).
- [Godot core types](https://docs.rs/gdnative/latest/gdnative/core_types/index.html) such as [`Color`](https://docs.rs/gdnative/latest/gdnative/core_types/struct.Color.html), [`Aabb`](https://docs.rs/gdnative/latest/gdnative/core_types/struct.Aabb.html), [`Transform2D`](https://docs.rs/gdnative/latest/gdnative/core_types/type.Transform2D.html), [`Vector2`](https://docs.rs/gdnative/latest/gdnative/core_types/type.Vector2.html), etc.
- Godot classes such as `Node`, `Reference`, etc. which must be accessed via [`Ref<T>`](https://docs.rs/gdnative/latest/gdnative/struct.Ref.html) (you can't pass them by value, because Godot owns them).
- Any Rust struct that derives [`NativeClass`](https://docs.rs/gdnative/latest/gdnative/derive.NativeClass.html), through [`Instance<T>`](https://docs.rs/gdnative/latest/gdnative/nativescript/struct.Instance.html).


## How can I profile my code to measure performance?

There are a lot of ways to profile your code and they vary in complexity.

The simplest way is to use the `#[gdnative::profiled]` procedural macro that enables Godot to profile the attributed function. This option is useful for comparing performance gains when porting GDScript code to Rust or as a way to understand the relative "frame time" of your code. Don't forget to compile Rust code with `--release`!

For more information about the Godot profiler, please refer to the [official documentation](https://docs.godotengine.org/en/stable/tutorials/debug/debugger_panel.html?highlight=profiler#profiler).

In order for Godot to profile your function, all the following must be true:
- The function belongs to a struct that derives `NativeClass`.
- The function is included in an `impl` block that is attributed with the `#[methods]` attribute.
- The function is attributed with `#[export]` attribute.

As such, this method is _only_ useful for exported code and is subject to the Godot profiler's limitations, such as millisecond accuracy in profiler metrics.

The following example illustrates how your code should look when being profiled:

```rust
#[derive(NativeClass)]
#[inherit(Node)]
struct MyClass {}

#[methods]
impl MyClass {
    fn new(_owner: &Node) -> Self {
        Self {}
    }

    #[export]
    #[gdnative::profiled]
    fn my_profiled_function(&self, _owner: &Node) {
        // Any code in this function will be profiled.
    }
}
```

If you require insight into Rust code that is not exported to Godot, or would like more in-depth information regarding execution of your program, it will be necessary to use a Rust compatible profiler such as [puffin](https://crates.io/crates/puffin) or [perf](https://perf.wiki.kernel.org/index.php/Main_Page). These tools can be used to more accurately determine bottlenecks and the general performance of your Rust code.

**Note:** There are many more profilers than the ones listed and you should do your own research before selecting which one you wish to use.

For more information about profiling and other rust performance tips, please check out the [Rust performance book](https://nnethercote.github.io/perf-book/profiling.html).
