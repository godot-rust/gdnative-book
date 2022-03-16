# Migrating from godot-rust 0.9.x to 0.10.x

In version 0.10, a lot of work has gone into making the 
In version 0.9, we are attempting to resolve many long-standing problems in the older API. As a result, there are many breaking changes in the public interface. This is a quick guide to the new API for users that have used older versions.

## Minimum Supported Rust Version

This has been changed to 1.56, when migrating you will need to ensure that you are using **at least** Rust 1.56 or later or your projects may fail to build.

The expected Rust edition is also 2021, so you will need to ensure that you are currently using the 2021 Rust edition to continue working with this crate.


## Breaking API Changes

This is a brief overview of the API breaking changes:

Please refer to the [changelog](https://github.com/godot-rust/godot-rust/blob/master/CHANGELOG.md) for a comprehensive list.

### Changes to Modules

The module structure has been simplified to ensure there is only 1 module per symbol.
- `nativescript` has been renamed to `export`
- `nativescript::{Instance, RefInstance}` has been moved to `object`
- `bindings` module has been removed
- `macrosgodot_gdnative_init`, `godot_gdnative_terminate`, `godot_nativescript_init`, `godot_site` have been removed from the prelude
- Unnecessary nested modules have also been removed, if you were depending upon the exact path, you will need to use the new path

### Changes to Core Types

- The euclid vector library has been removed and replaced with [glam](https://docs.rs/glam/latest/glam/)
- Variant has a redesigned conversion API (#819)
- Various type names have been changed for clarity

| Old Type Name | New Type Name|
| --- | --- |
| RefInstance | TInstance |
| RefKind | Memory |
| ThreadAccess | Ownership |
| TypedArray | PoolArray |
| Element | PoolElement |
| SignalArgument | SignalParam |
| Point2 | Vector2 |
| Size2 | Vector2 |

- Geometric types have been renamed for API consistency from x, y, z to a, b, c
- The following methods have been renamed:

| Old Method | New Method |
| --- | --- |
| {String,Variant}::forget() | leak() |
| Color::{rgb,rgba}() | {from_rgb,from_rgba}() |
| Rid::is_valid() | is_occupied() |
| Basis::to_scale() | scale() |
| Basis::from_elements() | from_rows() |
| Transform2D::from_axis_origin() | from_basis_origin() |

- The following deprecated symbols have been removed
  - Reference::init_ref() (unsound)
  - ClassBuilder::add_method(), 
  - add_method_advanced(), 
  - add_method_with_rpc_mode()
  - ScriptMethod
  - ScriptMethodFn 
  - ScriptMethodAttributes

- The following methods were removed due to being redundant
  - access methods for VariantArray<Shared>
  - Basis::invert()
  - Basis::orthonormalize()
  - Basis::rotate()
  - Basis::tdotx()
  - Basis::tdoty()
  - Basis::tdotz()
  - Rid::operator_less()

### Changes to Procedural Macros

- `#[inherit]` is now optional and defaults to `Reference` instead of `Node`

### Ownership Changes

- Instance and TInstance now use Own=Shared by default and some adjustments may be necessary.

## New features

While these are not breaking changes, the following may be useful to consider when migrating, particularly if yuou were previously using a custom solution for either of the following.

- [serde](https://serde.rs/) is now supported for VariantDispatch and all core types
- Async Foundations have been completed so you can now make use of Rust Async runtimes with Godot more easily. We have a recipe for using [async with the tokio runtime](../recipes/async-tokio.md)
- custom godot builds are now supported. The advanced guide for [custom godot](./custom-godot.md) has been updated accordingly.

## Migrations

### Variant/VariantDispatch

If you were using the `Variant::from_*` methods, this is no longer necessary and the functions no longer exist.

In 0.9.x you would need to use the specific constructor

```rust
let variant = Variant::from_i64(42);
let variant = Variant::from_f64(42.0);
let variant2 = Variant::from_object(object);
```

In 0.10.x, `new()` is sufficient for any type that implements ToVarant 

```rust
let variant = Variant::new(42);
let variant = Variant::new(42.0);
let variant2 = Variant::new(object);
```

### Transforms

Previously Transforms were defined by the matrix identities such as m11, m12, now they are referred by the vector name for consistency.

For example: When creating a identity Transform2D in 0.9.x it would be:

```rust
let tform = Transform2D::new(1.0, 0.0, 0.0, 1.0, 1.0, 1.0)
```

In 0.10.x it is defined as

```rust
Transform2D::from_basis_origin(
    Vector2::new(1.0, 0.0),
    Vector2::new(0.0, 1.0),
    Vector2::new(1.0, 1.0),
  )
)
```

## Vector Types

The underlying vector library as well as the implementation. In 0.9.x, many of the goemetrics previously were thinnly wrapping a separate libary. This lead to several different wrapping classes such as Point2, Size2 being removed as they were essentially an alias to Vector2. In addition other Geometric types -- such as Rect2, Quat, Transform, Plane -- have all been changed and some convenience functions may not be available anymore depending upon the struct.

For example: Rect2 no longer has width() or height()

### ClassBuilder

The [`ClassBuilder`](https://docs.rs/gdnative/latest/gdnative/prelude/struct.ClassBuilder.html) has been extended to use the builder pattern when registering signals and properties.

In 0.9.x registering a signal would look like the following:

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

In 0.10.0 this changes to

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

  // If you only need a default value, you can also register a signal like this
  builder.signal("signal3")
    .with_param_default("myArg", 42.0.to_variant())
    .done()
}
```

### Server Singletons

Godot's Server Singletons have received a safety overhaul and many functions that were previously marked "safe" now are marked unsafe. This change was necessary as it was identified that it was relatively trivial to create Undefined Behavior.