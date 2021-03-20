# FAQ

## Avoiding a `BorrowFailed` error on method call

**Question**

What is the `BorrowFailed` error and why do I keep getting it? I'm only trying to call another method that takes `&mut self` while holding one!

**Answer**

In Rust, [there can only be *one* `&mut` reference to the same memory location at the same time](https://docs.rs/dtolnay/0.0.9/dtolnay/macro._02__reference_types.html). To enforce this while making simple use cases easier, the bindings make use of [interior mutability](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html). This works like a lock: whenever a method with `&mut self` is called, it will try to obtain a lock on the `self` value, and hold it *until it returns*. As a result, if another method that takes `&mut self` is called in the meantime for whatever reason (e.g. signals), the lock will fail and an error (`BorrowFailed`) will be produced.

It's relatively easy to work around this problem, though: Because of how the user-data container works, it can only see the outermost layer of your script type - the entire structure. This is why it's stricter than what is actually required. If you run into this problem, you can [introduce finer-grained interior mutability](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html) in your own type, and modify the problematic exported methods to take `&self` instead of `&mut self`.

## Passing additional arguments to a class constructor

**Question**

A native script type needs to implement `fn new(owner: &Node) -> Self`.  
Is it possible to pass additional arguments to `new`?

**Answer**

Unfortunately this is currently a general limitation of GDNative (see [related issue](https://github.com/godotengine/godot/issues/23260)).

As a result, a common pattern to work-around the limitation is to use explicit initialization methods. For instance:

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


## Static methods

**Question**

In GDScript, classes can have static methods.
However, when I try to omit `self` in the exported method signature, I'm getting a compile error.
How can I implement a static method?

**Answer**

This is another limitation of GDNative -- static methods are not supported in general.

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

[`Aether`](https://docs.rs/gdnative/0.9/gdnative/prelude/struct.Aether.html) is a special user-data wrapper intended for zero-sized types, that does not perform any allocation or synchronization at runtime.

The type needs to be instantiated somewhere on GDScript level.
Good places for instantiation are for instance:

- as a member of a long-living util object,
- as a [singleton auto-load object](https://docs.godotengine.org/en/stable/getting_started/step_by_step/singletons_autoload.html).


## Converting a Godot type to the underlying Rust type

**Question**

I have a method that takes an argument `my_object` as a `Variant`.
I know that this object has a Rust native script attached to it, called say `MyObject`.
How can I access the Rust type given the Variant?

**Answer**

This conversion can be accomplished by casting the `Variant` to a `Ref`, and then to an `Instance` or `RefInstance`, and mapping over it to access the Rust data type:

```rust
#[methods]
impl AnotherNativeScript {

    #[export]
    pub fn method_accepting_my_object(&self, _owner: &Object, my_object: Variant) {
        // 1. Cast Variant to Ref of associated Godot type, and convert to TRef.
        let my_object = unsafe {
            my_object
                .try_to_object::<Object>()
                .expect("Failed to convert my_object variant to object")
                .assume_safe()
        };
        // 2. Obtain a RefInstance.
        let my_object = my_object
            .cast_instance::<MyObject>()
            .expect("Failed to cast my_object object to instance");
        // 3. Map over the RefInstance to extract the underlying user data.
        my_object
            .map(|my_object, _owner| {
                // Now my_object is of type MyObject.
            })
            .expect("Failed to map over my_object instance");
    }

}

```

## Auto-completion with rust-analyzer

**Question**

`godot-rust` generates most of the gdnative type's code at compile-time. Editors using [rust-analyzer](https://github.com/rust-analyzer/rust-analyzer) struggle to autocomplete those types:

![no-completion](images/no-completion.png)

**Answer**

People [reported](https://github.com/rust-analyzer/rust-analyzer/issues/5040) similar issues and found that switching on the `"rust-analyzer.cargo.loadOutDirsFromCheck": true` setting fixed it:

![completion](images/completion.png)


## Auto-completion with IntelliJ Rust Plugin

**Question**

Similar to rust-analyzer, IntelliJ-Family IDEs struggle to autocomplete gdnative types generated at compile-time.

**Answer**

There are two problems preventing autocompletion of gdnative types in IntelliJ-Rust.

First, the features necessary are (as of writing) considered experimental and must be enabled. Press `shift` twice to open the find all dialog and type `Experimental features...` and click the checkbox for `org.rust.cargo.evaluate.build.scripts`.  Note that `org.rust.cargo.fetch.out.dir` will also work, but is known to be less performant and may be phased out.

Second, the bindings files generated (~8mb) are above the 2mb limit for files to be processed. As [reported](https://github.com/intellij-rust/intellij-rust/issues/6571#) you can increase the limit with the steps below.
* open custom VM options dialog (`Help` | `Find Action` and type `Edit Custom VM Options`)
* add `-Didea.max.intellisense.filesize=limitValue` line where `limitValue` is desired limit in KB, for example, 10240. Note, it cannot be more than 20 MB.
* restart IDE