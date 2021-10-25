# Exported properties

Like methods, properties can be exported. The `#[property]` attribute above a field declaration makes the field available to Godot, with its name and type.

In the previous example, we could replace the `count_enemies()` method with a property `enemy_count`.
```rust
#[derive(NativeClass)]
#[inherit(Node)]
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

## Export options

The `#[property]` attribute can accept a several options to refine the export behavior.

You can specify default property value with the following argument:

```rust
#[property(default = 10)]
enemy_count: i32,
```

If you need to hide this property in Godot editor, use `no_editor` option:

```rust
#[property(no_editor)]
enemy_count: i32,
```

## Property access hooks

You can add hooks to call methods before and after property value was retrieved or changed.

Note that unlike [GDScript's `setget` keyword](https://docs.godotengine.org/en/3.3/getting_started/scripting/gdscript/gdscript_basics.html?#setters-getters), this does _not_ register a custom setter or getter. Instead, it registers a callback which is invoked _before or after_ the set/get occurs, and lacks both parameter and return value.

```rust
#[derive(NativeClass, Default)]
#[inherit(Node)]
struct GodotApi {
    // before get
    #[property(before_get = "Self::before_get")]
    prop_before_get: i32,

    // before set
    #[property(before_set = "Self::before_set")]
    prop_before_set: i32,

    // after get
    #[property(after_get = "Self::after_get")]
    prop_after_get: i32,

    // after set
    #[property(after_set = "Self::after_set")]
    prop_after_set: i32,
}

impl GodotApi {
    fn new(_owner: &Node) -> Self {
        Self::default()
    }

    fn before_get(&self, _owner: TRef<Node>) {
        godot_print!("Before get");
    }

    fn before_set(&self, _owner: TRef<Node>) {
        godot_print!("Before set");
    }

    fn after_get(&self, _owner: TRef<Node>) {
        godot_print!("After get");
    }

    fn after_set(&self, _owner: TRef<Node>) {
        godot_print!("After set");
    }
}
```
