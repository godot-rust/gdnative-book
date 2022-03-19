# Migrating from godot-rust 0.9.x to 0.10.x

Version 0.10 implements many improvements to ergonomics, naming consistency and bugfixes. Tooling and CI has been majorly overhauled, providing fast 
feedback cycles, higher confidence and easier-to-read documentation.

This guide outlines what actions users of godot-rust need to take to update their code.


## Minimum supported Rust version

The MSRV has been increased to 1.56. When migrating, you will need to ensure that you are using **at least** Rust 1.56 or later or your projects may fail to build.

We use the Rust 2021 edition; in your own code you may use any edition.


## Breaking API changes

This is a brief overview of the smaller breaking changes in the library API. Please refer to the [changelog](https://github.com/godot-rust/godot-rust/blob/master/CHANGELOG.md) for a comprehensive list. 

More sophisticated breaking changes are explained further down in section [_Migrations_](#Migrations).

### Changes to modules

The module structure has been simplified to ensure there is only one module per symbol:
- Module `nativescript` has been renamed to `export`.
- Types `nativescript::{Instance, RefInstance}` have been moved to `object`.
- Less often used macros `godot_gdnative_init`, `godot_gdnative_terminate`, `godot_nativescript_init`, `godot_site` have been removed from the prelude.
- Unnecessarily nested modules have also been removed. If you were depending upon the exact path, you will need to use the new path.

### Changes to core types

- The euclid vector library has been removed and replaced with [glam](https://docs.rs/glam/latest/glam/).

- `Variant` has a redesigned conversion API.

- Matrix types -- `Transform2D`, `Transform` and `Basis` -- have had their basis vectors renamed from `x/y/z` to `a/b/c`, to avoid confusion with the `x/y/z` vector components.

- The following deprecated symbols have been removed:
  - `Reference::init_ref`(unsound)
  - `ClassBuilder::add_method`, `add_method_advanced`, `add_method_with_rpc_mode`
  - `ScriptMethod`
  - `ScriptMethodFn` 
  - `ScriptMethodAttributes`

- The following methods were removed due to being redundant:
  - unsafe access methods for `VariantArray<Shared>` (available in `VariantArray<Unique>`)
  - `Basis::invert`
  - `Basis::orthonormalize`
  - `Basis::rotate`
  - `Basis::tdotx`
  - `Basis::tdoty`
  - `Basis::tdotz`
  - `Rid::operator_less`
  - `StringName::operator_less`


Various type names have been changed to improve clarity and consistency:

| Old Type Name    | New Type Name |
|------------------|---------------|
| `RefInstance`    | `TInstance`   |
| `RefKind`        | `Memory`      |
| `ThreadAccess`   | `Ownership`   |
| `TypedArray`     | `PoolArray`   |
| `Element`        | `PoolElement` |
| `SignalArgument` | `SignalParam` |
| `Point2`         | `Vector2`     |
| `Size2`          | `Vector2`     |

The following methods have been renamed:

| Old Method                      | New Method             |
|---------------------------------|------------------------|
| `{String,Variant}::forget`      | `leak`                 |
| `Color::{rgb,rgba}`             | `{from_rgb,from_rgba}` |
| `Rid::is_valid`                 | `is_occupied`          |
| `Basis::to_scale`               | `scale`                |
| `Basis::from_elements`          | `from_rows`            |
| `Transform2D::from_axis_origin` | `from_basis_origin`    |
| `StringName::get_name`          | `to_godot_string`      |


### Changes to procedural macros

- `#[inherit]` is now optional and defaults to `Reference` instead of `Node`.
- `#[property(before_set)]` and its siblings are replaced with `#[property(set)]` etc.; see below. 

### Ownership Changes

- `Instance` and `TInstance` now use `Own=Shared` by default. Some adjustments to your assumptions should be re-evaluated as needed.


## New features

In addition to new functionality outlined here, it can be interesting to check the _Added_ section in the changelog.

### Cargo features

While these are not breaking changes, the following may be useful to consider when migrating, particularly if you were previously using a custom solution for either of the following:

- [serde](https://serde.rs/) is now supported for `VariantDispatch` and types in the `core_types` module.
- Async Foundations have been completed, so you can now make use of Rust `async` runtimes with Godot more easily. We have a recipe for using [async with the tokio runtime](../recipes/async-tokio.md).
- Custom Godot builds are now supported. The advanced guide for [Custom Godot](./custom-godot.md) has been updated accordingly.

### Custom property exports

In godot-rust 0.9, it was necessary to manually register properties using the class builder such as the following:

```rust
#[derive(NativeClass)]
#[inherit(Reference)]
struct Foo {
    #[property]
    bar: i64,
}

#[methods] 
impl Foo {
  fn register_properties(builder: &ClassBuilder<Foo>) {
    builder
        .add_property::<String>("bar")
        .with_getter(get_bar)
        .with_setter(set_bar)
        .with_default(0)
        .done();
  }
  #[export]
  fn set_bar(&mut self, _owner: &Reference, value: i64) {
      self.bar = value;
  }

  #[export]
  fn get_bar(&mut self, _owner: &Reference) -> i64 {
      self.bar
  }
}
```

In 0.10, this can be automated with the `#[property]` procedural macro, such as the following:
```rust
#[derive(NativeClass)]
#[inherit(Reference)]
struct Foo {
    #[property(name = "bar", set = "set_bar", get = "get_bar", default = 0)]
    bar: i64,
}

#[methods] 
impl Foo {
    #[export]
    fn set_bar(&mut self, _owner: &Reference, value: i64) {
        self.bar = value;
    }

    #[export]
    fn get_bar(&mut self, _owner: &Reference) -> i64 {
        self.bar
    }
}
```

### `VariantDispatch`

`VariantDispatch` is an newly introduced type in godot-rust 0.10. This enum lets you treat `Variant` in a more rust-idiomatic way, e.g. by pattern-matching its contents:

```rust
let variant = 42.to_variant();

let number_as_float = match variant.dispatch() {
    VariantDispatch::I64(i) => i as f64,
    VariantDispatch::F64(f) => f,
    _ => panic!("not a number"),
};

approx::assert_relative_eq!(42.0, number_as_float);
```


## Migrations

This section elaborates on APIs with non-trivial changes and guides you through the process of updating your code.

### `Variant`

If you were using the `Variant::from_*` methods, those no longer exist.

In 0.9.x you would need to use the specific constructor, such as the following:

```rust
let variant = Variant::from_i64(42);
let variant = Variant::from_f64(42.0);
let variant2 = Variant::from_object(object);
```

In 0.10.x, `new()` is sufficient for any type that implements `ToVariant`, such as the following:

```rust
let variant = Variant::new(42);
let variant = Variant::new(42.0);
let variant2 = Variant::new(object);
```

When converting from a variant to a Rust type, it previously was necessary to do the following:

```rust
let integer64 = i64::from_variant(&variant_i64).unwrap();
let float64 = f64::from_variant(&variant_f64).unwrap();
let object = ObjectType::from_variant(&variant_object).unwrap();
```

In 0.10.x, you can now cast your variants by using the `to()` function on `FromVariant`-enabled types, such as the following:

```rust
// Note: If the compiler can infer your type, the turbofish `::<T>` is optional
let integer64 = variant.to::<i64>().unwrap();
let float64 = variant.to::<f64>().unwrap();
let object = variant.to_object::<ObjectType>().unwrap(); // returns Ref<ObjectType>
```


### Transforms

Previously, transforms were defined by the matrix identities such as `m11`, `m12`; now, they are referred by the vector name for consistency.

For example: When creating an identity `Transform2D` in 0.9.x, you would create it using the following code:

```rust
let tform = Transform2D::new(1.0, 0.0, 0.0, 1.0, 1.0, 1.0);
```

In 0.10.x you now need to create it using `from_basis_origin` and use `a`, `b`, and `origin` vectors, such as the following:

```rust
let tform = Transform2D::from_basis_origin(
    Vector2::new(1.0, 0.0),
    Vector2::new(0.0, 1.0),
    Vector2::new(1.0, 1.0),
);
```

### Vector types

The underlying vector library as well as the implementation have been fundamentally replaced. In 0.9.x, many of the goemetric types were thinly wrapping a separate library. This led to several wrapping classes such as `Point2`, `Size2` being removed now. In addition, other geometric types -- for example `Rect2`, `Quat`, `Transform`, `Plane` -- have all been changed, and certain convenience functions may not be available anymore, depending upon the struct.

For example: `Rect2` no longer has `width()` or `height()`, but lets you directly access its `size.x` or `size.y` fields.

### `ClassBuilder`

The [`ClassBuilder`](https://docs.rs/gdnative/latest/gdnative/prelude/struct.ClassBuilder.html) type has been extended to use the builder pattern when registering signals and properties.

In 0.9, registering a signal would look like the following:

```rust
fn register_signals(builder: &ClassBuilder<Self>) {
  builder.add_signal(
    Signal {
      name: "signal1",
      args: &[],
    }
  );
  builder.add_signal(
    Signal {
      name: "signal2",
      args: &[SignalArgument {
          name: "myArg",
          default: 42.0.to_variant(),
          export_info: ExportInfo::new(VariantType::F64),
          usage: PropertyUsage::DEFAULT,
      }],
  });
}
```

In 0.10, this changes to:

```rust
fn register_signals(builder: &ClassBuilder<Self>) {
  builder.signal("signal1").done();
  
  builder.signal("signal2")
    .with_param_custom(
      SignalParam {
          name: "myArg",
          default: 42.0.to_variant(),
          export_info: ExportInfo::new(VariantType::F64),
          usage: PropertyUsage::DEFAULT,
      },
    ).done();

  // If you only need a default value, you can also register a signal like this:
  builder.signal("signal3")
    .with_param_default("myArg", 42.0.to_variant())
    .done()
}
```


### Server singletons

Godot's server singletons have received a safety overhaul. As a result, all functions that take one or more parameters of type `Rid` are now marked `unsafe` and thus require being used inside an `unsafe` block or `unsafe` function.

In 0.9.x, creating a canvas_item and attaching it to a parent would be done as follows:
```rust
let vs = unsafe { VisualServer::godot_singleton() };
let canvas = vs.canvas_create();
let ci = vs.canvas_item_create();
vs.canvas_item_set_parent(ci, canvas);
```

In 0.10.x, you now must wrap the `canvas_item_set_parent` function in an `unsafe` block, such as follows:

```rust
let vs = unsafe { VisualServer::godot_singleton() };
let canvas = vs.canvas_create();
let ci = vs.canvas_item_create();
unsafe {
  vs.canvas_item_set_parent(ci, canvas);
}
```

#### Additional UB notes

The reason for this change was due to [issue #836](https://github.com/godot-rust/godot-rust/issues/836) being raised. Developers were able to demonstrate that you could easily cause undefined behavior when using any function that accepted `Rid` as a parameter, such as the following:

```rust
let vs = unsafe { VisualServer::godot_singleton() };
let canvas = vs.canvas_create();
let vp = vs.viewport_create();
vs.canvas_item_set_parent(vp, canvas); // crashes immediately
vs.canvas_item_set_parent(canvas, canvas); // crashes at shutdown
```
