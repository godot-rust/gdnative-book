# Migrating from godot-rust 0.8.x

In version 0.9, we are attempting to resolve many long-standing problems in the older API. As a result, there are many breaking changes in the public interface. This is a quick guide to the new API for users that have used older versions.

## Module organization and naming

### Generated API types

Generated types now live under the `gdnative::api` module. This makes the top-level namespace easier to navigate in docs. If you have used glob imports like:

```rust
use gdnative::*;
```

..., you should change this to:

```rust
use gdnative::prelude::*;
use gdnative::api::*;
```

### Generated property accessors

Generated getters of properties are no longer prefixed with `get_`. This **does not** affect other methods that start with `get_` that are not getters, like `File::get_8`.

### Separation of core types and NativeScript code

Core types, like `VariantArray`, `Color`, and `Dictionary` are moved to the `gdnative::core_types` module, while NativeScript supporting code like `NativeClass`, `Instance` and `init` code are moved to the `gdnative::nativescript` module. Most of the commonly-used types are re-exported in the `prelude`, but if you prefer individual imports, the paths need to be changed accordingly.

### API enums

C enums in the API are now generated in submodules of `gdnative_bindings` named after their associated objects. Common prefixes are also stripped from the constant names. The constants are accessed as associated constants:

#### Example

In 0.8, you would write this:

```rust
use gdnative::*;

object.connect(
    "foo".into(),
    owner.to_object(),
    "_handle_foo".into(),
    VariantArray::new(),
    Object::CONNECT_DEFERRED,
);
```

In 0.9, this should now be written as:

```rust
use gdnative::prelude::*;
use gdnative::api::object::ConnectFlags;

object.connect("foo", owner, "_handle_foo", VariantArray, ConnectFlags::DEFERRED.into());
```

### Fixed typos

Typos in variant names of `VariantOperator` and `GodotError` are fixed. Change to the correct names if this breaks your code.

## Changes to derive macros

The `NativeScript` derive macro now looks for `new` instead of `_init` as the constructor.

### Example

In 0.8, you would write this:

```rust
#[derive(NativeClass)]
#[inherit(Object)]
pub struct Thing;

impl Thing {
    fn _init(_owner: Object) -> Self {
        Thing
    }
}
```

In 0.9, this should now be written as:

```rust
#[derive(NativeClass)]
#[inherit(Object)]
pub struct Thing;

impl Thing {
    // `owner` may also be `TRef<Object>`, like in exported methods.
    fn new(_owner: &Object) -> Self {
        Thing
    }
}
```

## Argument casts

Generated methods taking objects, `Variant`, and `GodotString` are now made generic using `impl Trait` in argument position. This make calls much less verbose, but may break inference for some existing code. If you have code that looks like:

```rust
let mut binds = VariantArray::new();
binds.push(&42.to_variant());
binds.push(&"foo".to_variant());
thing.connect(
    "foo".into(),
    Some(owner.to_object()),
    "bar".into(),
    binds,
    0,
)
```

..., you should change this to:

```rust
let mut binds = VariantArray::new();
binds.push(42);
binds.push("foo");
thing.connect("foo", owner, "bar", binds, 0);
```

### Explicit nulls

A side effect of accepting generic arguments is that inference became tricky for `Option`. As a solution for this, an explicit `Null` type is introduced for use as method arguments. To obtain a `Null` value, you may use either `GodotObject::null()` or `Null::null()`. You may also call `null` on specific types since it is a trait method for all API types, e.g. `Node::null()` or `Object::null()`.

For example, to clear the script on an object, instead of:

```rust
thing.set_script(None);
```

..., you should now write:

```rust
thing.set_script(Null::null());
```

This is arguably less convenient, but passing explicit nulls should be rare enough a use case that the benefits of having polymorphic arguments are much more significant.

## Object semantics

In 0.9, the way Godot objects are represented in the API is largely remodeled to closely match the behavior of Godot. For the sake of illustration, we'll use the type `Node` in the following examples.

In 0.8.x, bare objects like `Node` are either unsafe or reference-counted pointers. Some of the methods require `&mut` receivers, but there is no real guarantee of uniqueness since the pointers may be cloned or aliased freely. This restriction is not really useful for safety, and requires a lot of operations to be `unsafe`.

This is changed in 0.9 with the typestate pattern, which will be explained later. Now, there are three representations of Godot objects, with different semantics:

| Type | Terminology | Meaning |
| - | - | - |
| `&'a Node` | bare reference | reference assumed or guaranteed to be valid and uncontended during `'a` |
| `Ref<Node, Access>` | persistent reference | stored reference whose validity is not always known, depending on the typestate `Access` |
| `TRef<'a, Node, Access>` | temporary reference | reference assumed or guaranteed to be valid and uncontended during `'a`, with added typestate tracking |

Note that owned bared objects, like `Node`, no longer exist in the new API. They should be replaced with `Ref` or `TRef` depending on the situation:

- In persistent data structures, like `NativeScript` structs, use `Ref<Node, Access>`.
- When taking the `owner` argument, use `&Node` or `TRef<'a, Node, Shared>`. The latter form allows for more safe operations, like using as method arguments, thanks to the typestate.
- When taking objects from the engine, like as arguments other than `owner`, use `Ref<Node, Shared>`.
- When passing objects to the engine as arguments or return values, use `&Ref<Node, Shared>`, `Ref<Node, Unique>`, or `TRef<'a, Node, Shared>`.
- When passing temporary references around in internal code, use `TRef<'a, Node, Access>`.

All objects are also seen as having interior mutability in Rust parlance, which means that all API methods now take `&self` instead of `&mut self`. For more information about shared (immutable) and unique (mutable) references, see [Accurate mental model for Rust's reference types](https://docs.rs/dtolnay/0.0.9/dtolnay/macro._02__reference_types.html) by dtolnay.

Unlike the previous versions requiring ubiquitous `unsafe`s, the new API allows for clearer separation of safe and unsafe code in Rust. In general, `unsafe` is only necessary around the border between Godot and Rust, while most of the internal code can now be safe.

To convert a `Ref` to a `TRef`, use `Ref::assume_safe` or `Ref::as_ref` depending on the object type and typestate. To convert non-unique `TRef`s to `Ref`s, use `TRef::claim`. Bare references are usually obtained through `Deref`, and it's not recommended to use them directly.

For detailed information about the API, see the type-level documentation on `Ref` and `TRef`.

### Example

If you prefer to learn by examples, here is the simplified code for instancing child scenes in the dodge-the-creeps example in 0.8. Note how everything is `unsafe` due to `Node`s being manually-managed:

```rust
#[derive(NativeClass)]
#[inherit(Node)]
pub struct Main {
    #[property]
    mob: PackedScene,
}

#[methods]
impl Main {
    #[export]
    unsafe fn on_mob_timer_timeout(&self, owner: Node) {
        let mob_scene: RigidBody2D = instance_scene(&self.mob).unwrap();
        let direction = rng.gen_range(-PI / 4.0, PI / 4.0);
        mob_scene.set_rotation(direction);
        owner.add_child(Some(mob_scene.to_node()), false);
    }
}

unsafe fn instance_scene<Root>(scene: &PackedScene) -> Result<Root, ManageErrs>
where
    Root: gdnative::GodotObject,
{
    let inst_option = scene.instance(PackedScene::GEN_EDIT_STATE_DISABLED);

    if let Some(instance) = inst_option {
        if let Some(instance_root) = instance.cast::<Root>() {
            Ok(instance_root)
        } else {
            Err(ManageErrs::RootClassNotRigidBody2D(
                instance.name().to_string(),
            ))
        }
    } else {
        Err(ManageErrs::CouldNotMakeInstance)
    }
}
```

In 0.9, this can now be written as (with added type annotations and explanations for clarity):

```rust
#[derive(NativeClass)]
#[inherit(Node)]
pub struct Main {
    #[property]
    mob: Ref<PackedScene, Shared>,
}

#[methods]
impl Main {
    #[export]
    fn on_mob_timer_timeout(&self, owner: &Node) {
        // This assumes that the `PackedScene` cannot be mutated from
        // other threads during this method call. This is fine since we
        // know that no other scripts mutate the scene at runtime.
        let mob: TRef<PackedScene, Shared> = unsafe { self.mob.assume_safe() };

        // Call the internal function `instance_scene` using the `TRef`
        // produced by `assume_safe`. This is safe because we already
        // assumed the safety of the object.
        let mob_scene: Ref<RigidBody2D, Unique> = instance_scene(mob);

        // `mob_scene` can be used directly because it is a `Unique`
        // reference -- we just created it and there can be no other
        // references to it in the engine.
        let direction = rng.gen_range(-PI / 4.0, PI / 4.0);
        mob_scene.set_rotation(direction);

        // `mob_scene` can be passed safely to the `add_child` method
        // by value since it is a `Unique` reference.
        // Note that there is no need to cast the object or wrap it in an
        // `Option`.
        owner.add_child(mob_scene, false);

        // Note that access to `mob_scene` is lost after it's passed
        // to the engine. If you need to use it after doing that,
        // convert it into a `Shared` one with `into_shared().assume_safe()`
        // first.
    }
}

// The `instance_scene` function now expects `TRef` arguments, which expresses
// at the type level that this function expects a valid and uncontended
// reference during the call.
//
// Note that this makes the interface safe.
fn instance_scene<Root>(scene: TRef<PackedScene, Shared>) -> Ref<Root, Unique>
where
    // More static guarantees are now available for `cast` using the `SubClass`
    // trait. This ensures that you can only downcast to possible targets.
    Root: gdnative::GodotObject<RefKind = ManuallyManaged> + SubClass<Node>,
{
    // Instancing the scene is safe since `scene` is assumed to be safe.
    let instance = scene
        .instance(PackedScene::GEN_EDIT_STATE_DISABLED)
        .expect("should be able to instance scene");

    // `PackedScene::instance` is a generated API, so it returns
    // `Ref<_, Shared>` by default. However, we know that it actually creates
    // a new `Node` for us, so we can convert it to a `Unique` reference using
    // the `unsafe` `assume_unique` method.
    let instance = unsafe { instance.assume_unique() };

    // Casting to the `Root` type is also safe now.
    instance
        .try_cast::<Root>()
        .expect("root node type should be correct")
}
```

## Casting

The casting API is made more convenient with 0.9, using the `SubClass` trait. Casting is now covered by two generic methods implemented on all object reference types: `cast` and `upcast`. Both methods enforce cast validity statically, which means that the compiler will complain about casts that will always fail at runtime. The generated `to_*` methods are removed in favor of `upcast`.

### Examples

#### Casting a `Node` to `Object`

In 0.8, you would write either of:

- `node.to_object()`
- `node.cast::<Object>().unwrap()`

In 0.9, this should now be written as `node.upcast::<Object>()`. This is a no-op at runtime because `Node` is a subclass of `Object`.

#### Casting an `Object` to `Node`

The API for downcasting to concrete types is unchanged. You should still write `object.cast::<Node>().unwrap()`.

#### Casting to a type parameter

In 0.8, you would write this:

```rust
fn polymorphic_cast<T: GodotObject>(obj: Object) -> Option<T> {
    obj.cast()
}
```

In 0.9, casting to a type parameter is a bit more complicated due to the addition of static checks. You should now write:

```rust
fn polymorphic_cast<T, Access>(
    obj: TRef<Object, Access>,
) -> Option<TRef<T, Access>>
where
    T: GodotObject + SubClass<Object>,
    Access: ThreadAccess,
{
    obj.cast()
}
```

Note that this function is also polymorphic over the `Access` typestate, which is explained the the following section.

## Typestates

The typestate pattern is introduced in 0.9 to statically provide fine-grained reference safety guarantees depending on thread access state. There are three typestates in the API:

- `Shared`, for references that may be shared between multiple threads. `Shared` references are `Send` and `Sync`.
- `ThreadLocal`, for references that may only be shared on the current thread. `ThreadLocal` references are `!Send` and `!Sync`. This is because sending a `ThreadLocal` reference to another thread violates the invariant.
- `Unique`, for references that are **globally unique** (with the exception of specific engine internals like `ObjectDB`). `Unique` references are `Send` but `!Sync`. These references are always safe to use.

Users may convert between typestates using the `into_*` and `assume_*` methods found on various types that deal with objects or `Variant` collections with interior mutability (`Dictionary` and `VariantArray`). Doing so has no runtime cost. The documentation should be referred to when calling the unsafe `assume_*` methods to avoid undefined behavior.

Typestates are encoded in types as the `Access` type parameter, like the ones in `Ref<T, Access>` and `Dictionary<Access>`. These type parameters default to `Shared`.

The general guidelines for using typestates are as follows:

- References obtained from the engine are `Shared` by default. This includes unique objects returned by generated API methods, because there isn't enough information to tell them from others.
- Constructors return `Unique` references.
- `ThreadLocal` references are never returned directly. They can be created manually from other references, and can be used Godot objects and collections internally kept in Rust objects.

### Examples

#### Creating a `Dictionary` and returning it

In 0.8, you would write this:

```rust
#[export]
fn dict(&self, _owner: Reference) -> Dictionary {
    let mut dict = Dictionary::new();
    dict.insert(&"foo".into(), &42.into());
    dict.insert(&"bar".into(), &1337.into());
    dict
}
```

In 0.9, this should now be written as (with added type annotations and explanations for clarity):

```rust
#[export]
fn dict(&self, _owner: &Reference) -> Dictionary<Unique> {
    // `mut` is no longer needed since Dictionary has interior mutability in
    // Rust parlance
    let dict: Dictionary<Unique> = Dictionary::new();

    // It is safe to insert into the `dict` because it is unique. Also note how
    // the conversion boilerplate is no longer needed for `insert`.
    dict.insert("foo", 42);
    dict.insert("bar", 1337);

    // Owned `Unique` references can be returned directly
    dict
}
```

#### Using an object argument other than `owner`

In 0.8, you would write this:

```rust
#[export]
unsafe fn connect_to(&self, owner: Node, target: Object) {
    target.connect(
        "foo".into(),
        owner.to_object(),
        "_handle_foo".into(),
        VariantArray::new(),
        0,
    );
}
```

In 0.9, this should now be written as (with added type annotations and explanations for clarity):

```rust
#[export]
// The `Access` parameter defaults to `Shared` for `Ref`, so it can be omitted
// in method signatures.
// Here, we also need to use `TRef` as the owner type, because we need the
// typestate information to pass it into `connect`.
fn connect_to(&self, owner: TRef<Node>, target: Ref<Object>) {
    // Assume that `target` is safe to use for the body of this method.
    let target = unsafe { target.assume_safe() };
    // `TRef` references can be passed directly into methods without the need
    // for casts.
    target.connect("foo", owner, "_handle_foo", VariantArray::new(), 0);
}
```
