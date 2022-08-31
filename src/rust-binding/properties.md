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

## Property get/set

Properties can register `set` and `get` methods to be called from Godot.

Default get/set functions can be registered as per the following example:

```rs

#[derive(NativeClass, Default)]
#[inherit(Node)]
struct GodotApi {
    // property registration
    // Note: This is actually equivalent to #[property]
    #[property(get, set)]
    prop: i32,
}
```

If you need custom setters and getters, you can set them in the `property` attribute such as in the following example:

```rust
#[derive(NativeClass)]
#[inherit(Node)]
struct HelloWorld {
    // property registration
    #[property(get = "Self::get", set = "Self::set")]
    prop: i32,
}

impl HelloWorld {
    fn new(_base: &Node) -> Self {
        HelloWorld { prop: 0i32 }
    }
}

#[methods]
impl HelloWorld {
    fn get(&self, _base: TRef<Node>) -> i32 {
        godot_print!("get() -> {}", &self.prop);
        self.prop
    }

    fn set(&mut self, _base: TRef<Node>, value: i32) {
        godot_print!("set({})", &value);
        self.prop = value;
    }
}
```

### Note: `get` vs `get_ref`

There are two ways to return the property.
- `get` will return a value of `T` which _must_ result in the value being cloned. 
- `get_ref` must point to a function that returns `&T`, this is useful when working with large data that would be very expensive to copy unnecessarily.

Modifying the previous example accordingly results in the following:

```rust
#[derive(NativeClass)]
#[inherit(Node)]
struct GodotApi {
    // property registration
    #[property(get_ref = "Self::get", set = "Self::set")]
    prop: String,
}

impl GodotApi {
    fn new(_base: &Node) -> Self {
        GodotApi { prop: String::new() }
    }
}

#[methods]
impl GodotApi {
    fn get(&self, _base: TRef<Node>) -> &String {
        godot_print!("get() -> {}", &self.prop);
        &self.prop
    }

    fn set(&mut self, _base: TRef<Node>, value: String) {
        godot_print!("set({})", &value);
        self.prop = value;
    }
}
```

## Manual property registration

For cases not covered by the `#[property]` attribute, it may be necessary to manually register the properties instead.

This is often the case where custom hint behavior is desired for primitive types, such as an Integer value including an `IntEnum` hint.

To do so, you can use the [`ClassBuilder`](https://docs.rs/gdnative/latest/gdnative/prelude/struct.ClassBuilder.html) -- such as in the following examples -- to manually register each property and customize how they interface in the editor.

```rust

#[derive(NativeClass)]
#[inherit(Node)]
#[register_with(Self::register_properties)]
#[no_constructor]
pub struct MyNode {
    number: i32,
    number_enum: i32,
    float_range: f64,
    my_filepath: String,
}

#[methods]
impl MyNode {
    fn register_properties(builder: &ClassBuilder<MyNode>) {
        use gdnative::export::hint::*;
        // Add a number with a getter and setter. 
        // (This is the equivalent of adding the `#[property]` attribute for `number`)
        builder
            .property::<i32>("number")
            .with_getter(Self::number_getter)
            .with_setter(Self::number_setter)
            .done();

        // Register the number as an Enum
        builder
            .property::<i32>("number_enum")
            .with_getter(move |my_node: &MyNode, _base: TRef<Node>| my_node.number_enum)
            .with_setter(move |my_node: &mut MyNode, _base: TRef<Node>, new_value| my_node.number_enum = new_value)
            .with_default(1)
            .with_hint(IntHint::Enum(EnumHint::new(vec!["a".to_owned(), "b".to_owned(), "c".to_owned(), "d".to_owned()])))
            .done();

        // Register a floating point value with a range from 0.0 to 100.0 with a step of 0.1
        builder
            .property::<f64>("float_range")
            .with_getter(move |my_node: &MyNode, _base: TRef<Node>| my_node.float_range)
            .with_setter(move |my_node: &mut MyNode, _base: TRef<Node>, new_value| my_node.float_range = new_value)
            .with_default(1.0)
            .with_hint(FloatHint::Range(RangeHint::new(0.0, 100.0).with_step(0.1)))
            .done();

        // Manually register a string as a file path for .txt and .dat files.
        builder
            .property::<String>("my_filepath")
            .with_ref_getter(move |my_node: &MyNode, _base: TRef<Node>| &my_node.my_filepath)
            .with_setter(move |my_node: &mut MyNode, _base: TRef<Node>, new_value: String| my_node.my_filepath = new_value)
            .with_default("".to_owned())
            .with_hint(StringHint::File(EnumHint::new(vec!["*.txt".to_owned(), "*.dat".to_owned()])))
            .done();
    }
    fn number_getter(&self, _base: TRef<Node>) -> i32 {
        self.number
    }

    fn number_setter(&mut self, _base: TRef<Node>, new_value: i32) {
        self.number = new_value
    }
}
```

## `Property<T>` and when to use it

Sometimes it can be useful to expose a value as a property instead of as a function. Properties of this type serve as a _marker_ that can be registered with Godot and viewed in the editor without containing any data in Rust.

This can be useful for data (similar to the first sample) where the count serves more as a property of `enemies` rather than as its own distinct data, such as the following: 

```rs
struct Enemy {
    // Enemy Data
}
#[derive(NativeClass)]
struct GodotApi {
    enemies: Vec<Enemy>,
    // Note: As the property is a "marker" property, this will never be used in code.
    #[allow(dead_code)]
    #[property(get = "Self::get_size")]
    enemy_count: Property<u32>,
}

#[methods]
impl GodotApi {
    //...

    fn get_size(&self, _base: TRef<Reference>) -> u32 {
        self.enemies.len() as u32
    }
}
```