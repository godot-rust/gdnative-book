# Exported properties

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
