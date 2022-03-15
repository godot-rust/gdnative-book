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

