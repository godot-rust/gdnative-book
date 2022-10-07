# Migrating from godot-rust 0.10.x to 0.11

Version 0.11 is a relatively small release, primarily adding support for the latest stable Godot version, 3.5.1 at the time.

Due to some breaking changes in the GDNative API, this is a SemVer-breaking release, but in practice you can reuse your 0.10.x
based code with near-zero changes. This migration guide is correspondingly brief.

For a detailed list of changes, refer to the [changelog](https://github.com/godot-rust/godot-rust/blob/master/CHANGELOG.md).

## Supported Godot version

The underlying Godot version changes from **3.4** (for gdnative 0.10) to **3.5.1**.

Note that we explicitly recommend Godot 3.5.1 and not just 3.5; the reason being that GDNative had minor (but breaking) changes
for this patch release.

If you want to keep using Godot 3.4 with latest godot-rust (or any Godot version for that matter), take a look at the 
[Cargo feature `custom-godot`](https://godot-rust.github.io/docs/gdnative/#feature-flags).


## Minimum supported Rust version

The MSRV has been increased to **1.63**.  
If you have an older toolchain, run `rustup update`.


## API Changes

### Method export syntax

Up until gdnative 0.10.0, methods need to be exported using the `#[export]` syntax.
If you did not need that parameter, the convention was to prefix it with `_`:

```rust
#[export]
fn _ready(&mut self, owner: &Node, delta: f32) {...}

#[export]
fn _process(&mut self, _owner: &Node, delta: f32) {...}
```

This changes starting from 0.10.1: `#[method]` is the new attribute for the method, and `#[base]` annotates
the parameter `base` (previously called `owner`). If the base parameter is not needed, simply omit it.

`base` still needs to be in 2nd position, immediately after the receiver (`&self`/`&mut self`).

```rust
#[method]
fn _ready(&mut self, #[base] base: &Node, delta: f32) {...}

#[method]
fn _process(&mut self, delta: f32) {...}
```

The old syntax is deprecated and stays valid throughout 0.11, so there is no urge to migrate.

### Removals

The constructor `Transform2D::from_rotation_translation_scale()` was removed, due to its unintuitive semantics.
It has been replaced with `from_scale_rotation_origin()`.

Symbols already deprecated in 0.10 have also been removed, with the exception of `#[export]`. In particular, the
type aliases `RefInstance` (now `TInstance`) and `TypedArray` (now `PoolArray`) no longer exist.
