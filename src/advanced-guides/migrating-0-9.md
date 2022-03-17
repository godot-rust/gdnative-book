# Migrating from godot-rust 0.9.x to 0.10.x

In version 0.10, many improvements to ergonomics, naming consistency, and resolving a number of bugs have been made. This has resulted in a number of the underlying 
## Minimum Supported Rust Version

This has been changed to 1.56, when migrating you will need to ensure that you are using **at least** Rust 1.56 or later or your projects may fail to build.

You may use any edition of Rust that supports 1.56, at the time of writing this is 2018 Edition and 2021 edition.

## Breaking API Changes

This is a brief overview of the API breaking changes:

Please refer to the [changelog](https://github.com/godot-rust/godot-rust/blob/master/CHANGELOG.md) for a comprehensive list.

### Changes to Modules

The module structure has been simplified to ensure there is only 1 module per symbol.
- `nativescript` has been renamed to `export`
- `nativescript::{Instance, RefInstance}` has been moved to `object`
- `macrosgodot_gdnative_init`, `godot_gdnative_terminate`, `godot_nativescript_init`, `godot_site` have been removed from the prelude
- Unnecessary nested modules have also been removed, if you were depending upon the exact path, you will need to use the new path

### Changes to Core Types

- The euclid vector library has been removed and replaced with [glam](https://docs.rs/glam/latest/glam/).
- Variant has a redesigned conversion API.
- Various type names have been changed to improve clarity and consistency.

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

- Basis vector types -- Transform2D, Transform and Basis -- have had their properties renamed from 'x', 'y', 'z' to 'a', 'b', 'origin' for API consistency.
- The following methods have been renamed:

| Old Method | New Method |
| --- | --- |
| {String,Variant}::forget | leak |
| Color::{rgb,rgba} | {from_rgb,from_rgba} |
| Rid::is_valid | is_occupied |
| Basis::to_scale | scale |
| Basis::from_elements | from_rows |
| Transform2D::from_axis_origin | from_basis_origin |

- The following deprecated symbols have been removed:
  - Reference::init_ref (unsound)
  - ClassBuilder::add_method, 
  - add_method_advanced, 
  - add_method_with_rpc_mode
  - ScriptMethod
  - ScriptMethodFn 
  - ScriptMethodAttributes

- The following methods were removed due to being redundant:
  - access methods for VariantArray<Shared>
  - Basis::invert
  - Basis::orthonormalize
  - Basis::rotate
  - Basis::tdotx
  - Basis::tdoty
  - Basis::tdotz
  - Rid::operator_less

### Changes to Procedural Macros

- `#[inherit]` is now optional and defaults to `Reference` instead of `Node`.

### Ownership Changes

- Instance and TInstance now use Own=Shared by default and some adjustments to your assumptions should be re-evaluated as needed.

## New features

The following new features may be useful to incorporate into your new code. In addition, it may be necessary to 
While these are not breaking changes, the following may be useful to consider when migrating, particularly if yuou were previously using a custom solution for either of the following.

- [serde](https://serde.rs/) is now supported for `VariantDispatch` and all `core_types`.
- Async Foundations have been completed so you can now make use of Rust Async runtimes with Godot more easily. We have a recipe for using [async with the tokio runtime](../recipes/async-tokio.md)
- Custom Godot builds are now supported. The advanced guide for [Custom Godot](./custom-godot.md) has been updated accordingly.

### Custom Property Exports

In 0.9.x, it was necessary to manually register properties using the class builder such as the following:

```rust
#[derive(NativeClass)]
#[inherit(Reference)]
struct Foo {
    #[property]
    bar: i64,
}

#[methods] 
impl Foo {
  fn register_properties(builder: &ClassBuilder<GodotEgui>) {
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

In 0.10.0 this can be automated with the `#[property]` procedural macro, such as the following:
```rust
#[derive(NativeClass)]
#[inherit(Reference)]
struct Foo {
    #[property(name="bar", set = "set_bar", get = "get_bar", default = 0)]
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

## Migrations

### Variant/VariantDispatch

If you were using the `Variant::from_*` methods, this is no longer necessary and the functions no longer exist.

In 0.9.x you would need to use the specific constructor, such as the following:

```rust
let variant = Variant::from_i64(42);
let variant = Variant::from_f64(42.0);
let variant2 = Variant::from_object(object);
```

In 0.10.x, `new()` is sufficient for any type that implements ToVarant, such as the following:

```rust
let variant = Variant::new(42);
let variant = Variant::new(42.0);
let variant2 = Variant::new(object);
```

When converting from a variant to a a Rust type, it previously was necessary to do the following:

```rust
let integer64 = i64::from_variant(&variant_i64).unwrap();
let float64 = f64::from_variant(&variant_f64).unwrap();
let object = ObjectType::from_variant(&variant_object).unwrap();
```

In 0.10.x, you can now cast your variants by using the `to()` function, such as the following:

```rust
// Note: If the compiler cannot infer your type, you may need to use the turbofish `::<T>` to compile
let integer64 = variant_i64.to().unwrap();
let float64 = variant_f64.to().unwrap();
let object = variant_object.to().unwrap();
```

### Transforms

Previously Transforms were defined by the matrix identities such as m11, m12, now they are referred by the vector name for consistency.

For example: When creating a identity Transform2D in 0.9.x, you would create it using the following code:

```rust
let tform = Transform2D::new(1.0, 0.0, 0.0, 1.0, 1.0, 1.0)
```

In 0.10.x you now need to create it using `from_basis_origin` and use `a`, `b`, and `origin` vectors, such as the following:

```rust
Transform2D::from_basis_origin(
    Vector2::new(1.0, 0.0),
    Vector2::new(0.0, 1.0),
    Vector2::new(1.0, 1.0),
  )
)
```

## Vector Types

The underlying vector library as well as the implementation. In 0.9.x, many of the goemetrics previously were thinnly wrapping a separate libary. This lead to several different wrapping classes such as Point2, Size2 being removed as they were aliases to Vector2. In addition other Geometric types -- such as Rect2, Quat, Transform, Plane -- have all been changed and some convenience functions may not be available anymore depending upon the struct.

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

Godot's Server Singletons have received a safety overhaul. As a result, all functions that take one or more parameters of type `Rid` are now `unsafe` functions and require being used inside an `unsafe` block or `unsafe` function.

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
#### Additional UB Notes

This was due to [Issue 836](https://github.com/godot-rust/godot-rust/issues/836) being raised that demonstrated that you can cause Undefined Behavior trivially by doing the following:

```rust
let vs = unsafe { VisualServer::godot_singleton() };
let canvas = vs.canvas_create();
let vp = vs.viewport_create();
vs.canvas_item_set_parent(vp, canvas); // crashes immediately
vs.canvas_item_set_parent(canvas, canvas); // crashes at shutdown
```
