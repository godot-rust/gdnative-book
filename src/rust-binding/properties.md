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

## Manual property registration

For cases not covered by the `#[property]` attribute, it may be necessary to manually register the properties instead.

This is often the case where custom hint behavior is desired for primitive types, such as an Integer value including an `IntEnum` hint.

To do so, you can use the [`ClassBuilder`](https://docs.rs/gdnative/latest/gdnative/prelude/struct.ClassBuilder.html) -- such as in the following examples -- to manually register each property and customize how they interface in the editor.

```rust
#[derive(NativeClass)]
#[inherit(gdnative::api::Node)]
#[register_with(Self::register_properties)]
pub struct MyNode {
    number: i32,
    number_enum: i32,
    float_range: f32,
    my_filepath: String,
}

#[gdnative::methods]
impl MyNode {
    fn register_properties(builder: &ClassBuilder<MyNode>) {
        use gdnative::nativescript::property::StringHint;
        // Add a number with a getter and setter. 
        // (This is the equivalent of adding the `#[property]` attribute for `number`)
        builder
            .add_property::<i32>("number")
            .with_getter(number_getter)
            .with_setter(numer_setter)
            .done();

        // Register the number as an Enum
        builder
            .add_property::<i32>("number_enum")
            .with_getter(move |my_node: &MyNode, _owner: TRef<Node>| my_node.number_enum)
            .with_setter(move |my_node: &mut MyNode, _owner: TRef<Node>, new_value| my_node.number_enum = new_value)
            .with_default(1)
            .with_hint(IntHint::Enum(EnumHint::new("a", "b", "c", "d")))
            .done();

        // Register a floating point value with a range from 0.0 to 100.0 with a step of 0.1
        builder
            .add_property::<f64>("float_range")
            .with_getter(move |my_node: &MyNode, _owner: TRef<Node>| my_node.float_range)
            .with_setter(move |my_node: &mut MyNode, _owner: TRef<Node>, new_value| my_node.float_range = new_value)
            .with_default(1.0)
            .with_hint(FloatHint::Range(RangeHint::new(0.0, 100.0).with_step(0.1)))
            .done();

        // Manually register a string as a file path for .txt and .dat files.
        builder
            .add_property::<String>("my_filepath")
            .with_getter(move |my_node: &MyNode, _owner: TRef<Node>| my_node.my_filepath.clone())
            .with_setter(move |my_node: &mut MyNode, _owner: TRef<Node>, new_value: String| my_node.my_filepath = new_value)
            .with_default("".to_owned())
            .with_hint(StringHint::File(EnumHint::new(vec!["*.txt".to_owned(), "*.dat".to_owned()])))
            .done();
    }
    fn number_getter(&self, _owner: TRef<Node>) -> i32 {
        self.number
    }

    fn number_setter(&mut self, _owner: TRef<Node>, new_value: i32) {
        self.number = new_value
    }
}
```
