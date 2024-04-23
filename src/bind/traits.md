# ToVariant, FromVariant and Export

As seen in [the previous section](./properties.md), the `#[property]` attribute of the [`NativeClass`](https://docs.rs/gdnative/latest/gdnative/derive.NativeClass.html) procedural macro is a powerful tool to automatically configure properties with Godot.

One constraint of the `#[property]` attribute is that it requires that all attributed property types implement `ToVariant`, `FromVariant` and `Export` in order to interface with Godot.

## ToVariant/FromVariant traits

In Godot all types inherit from Variant.

As per the [official Godot docs](https://docs.godotengine.org/en/stable/classes/class_variant.html), Variant is "The most important data type in Godot." This is a wrapper type that can store any [Godot Engine type](https://docs.godotengine.org/en/stable/classes/class_%40globalscope.html#enum-globalscope-variant-type)

The `ToVariant` and `FromVariant` are conversion traits that allow Rust types to be converted between these types. All properties must implement both `ToVariant` and `FromVariant` while exported methods require `FromVariant` to be implemented for optional parameters and `ToVariant` to be implemented for return types.

For many datatypes, it is possible to use the derive macros such as in the following example:

```rust
// Note: This struct does not implement `Export` and cannot be used as a property, see the following section for more information.
#[derive(ToVariant, FromVariant)]
struct Foo {
    number: i32,
    float: f32,
    string: String
}
```

For more information about how you can customize the behavior of the dervive macros, please refer to the official documentation for the latest information.

- [ToVariant](https://docs.rs/gdnative/latest/gdnative/core_types/trait.ToVariant.html)
- [FromVariant](https://docs.rs/gdnative/latest/gdnative/core_types/trait.FromVariant.html)

## Export Trait

The Godot editor retrieves property information from [Object::get_property_list](https://docs.godotengine.org/en/stable/classes/class_object.html#id2). To populate this data, `godot-rust` requires that the [`Export`](https://docs.rs/gdnative/latest/gdnative/nativescript/trait.Export.html) trait be implemented for each type Rust struct.

There are no derive macros that can be used for `Export` but many of the primitive types have it implemented by default.

To implement `Export` for the previous Rust data type, you can do so as in the following example:

```rust
// Note: By default `struct` will be converted to and from a Dictionary where property corresponds to a key-value pair.
#[derive(ToVariant, FromVariant)]
struct Foo {
    number: i32,
    float: f32,
    string: String
}

impl Export for Foo {
    // This type should normally be one of the types defined in [gdnative::export::hint](https://docs.rs/gdnative/latest/gdnative/export/hint/index.html).
    // Or it can be any custom type for differentiating the hint types.
    // In this case it is unused, so it is left as ()
    type Hint = ();
    fn export_info(hint: Option<Self::Hint>) -> ExportInfo {
        // As `Foo` is a struct that will be converted to a Dictionary when converted to a variant, we can just add this as the VariantType.
        ExportInfo::new(VariantType::Dictionary)
    }
}
```

## Case study: exporting Rust enums to Godot and back

A common challenge that many developers may encounter when using godot-rust is that while [Rust enums](https://doc.rust-lang.org/std/keyword.enum.html) are [Algebraic Data Types](https://en.wikipedia.org/wiki/Algebraic_data_type), [Godot enums](https://docs.godotengine.org/en/stable/getting_started/scripting/gdscript/gdscript_basics.html#enums) are constants that correspond to integer types.

By default, Rust enums are converted to a Dictionary representation. Its keys correspond to the name of the enum variants, while the values correspond to a Dictionary with fields as key-value pairs.

For example:

```rust
#[derive(ToVariant, FromVariant)]
enum MyEnum {
    A,
    B { inner: i32 },
    C { inner: String }
}
```

Will convert to the following dictionary:

```gdscript
# MyEnum::A
"{ "A": {} }
# MyEnum::B { inner: 0 }
{ "B": { "inner": 0 } }
# MyEnum::C { inner: "value" }
{ "C": {"inner": "value" } }
```

As of writing (gdnative 0.9.3), this default case is not configurable. If you want different behavior, it is necessary to implement `FromVariant` and `Export` manually for this data-type.

### Case 1: Rust Enum -> Godot Enum

Consider the following code:

```rust
enum MyIntEnum {
    A=0, B=1, C=2,
}

#[derive(NativeClass)]
#[inherit(Node)]
#[no_constructor]
struct MyNode {
    #[property]
    int_enum: MyIntEnum
}
```

This code defines the enum `MyIntEnum`, where each enum value refers to an integer value.

Without implementing the `FromVariant` and `Export` traits, attempting to export `MyIntEnum` as a property of `MyNode` will result in the following error:

```sh
the trait bound `MyIntEnum: gdnative::prelude::FromVariant` is not satisfied
   required because of the requirements on the impl of `property::accessor::RawSetter<MyNode, MyIntEnum>` for `property::accessor::invalid::InvalidSetter<'_>`2

the trait bound `MyIntEnum: Export` is not satisfied
    the trait `Export` is not implemented for `MyIntEnum`
```

This indicates that `MyIntEnum` does not have the necessary traits implemented for `FromVariant` and `Export`. Since the default derived behavior may not be quite what we want, we can implement this with the following:

```rust
impl FromVariant for MyIntEnum {
    fn from_variant(variant: &Variant) -> Result<Self, FromVariantError> {
        let result = i64::from_variant(variant)?;
        match result {
            0 => Ok(MyIntEnum::A),
            1 => Ok(MyIntEnum::B),
            2 => Ok(MyIntEnum::C),
            _ => Err(FromVariantError::UnknownEnumVariant {
                variant: "i64".to_owned(),
                expected: &["0", "1", "2"],
            }),
        }
    }
}

impl Export for MyIntEnum {
    type Hint = IntHint<u32>;

    fn export_info(_hint: Option<Self::Hint>) -> ExportInfo {
        Self::Hint::Enum(EnumHint::new(vec![
            "A".to_owned(),
            "B".to_owned(),
            "C".to_owned(),
        ]))
        .export_info()
    }
}

```

After implementing `FromVariant` and `Export`, running `cargo check` would result in the following additional error:

```sh
the trait bound `MyIntEnum: gdnative::prelude::ToVariant` is not satisfied
the trait `gdnative::prelude::ToVariant` is not implemented for `MyIntEnum`
```

If the default implementation were sufficient, we could use `#[derive(ToVariant)]` for `MyIntEnum` or implement it manually with the following code:

```rust
use gdnative::core_types::ToVariant;
impl ToVariant for MyIntEnum {
    fn to_variant(&self) -> Variant {
        match self {
            MyIntEnum::A => { 0.to_variant() },
            MyIntEnum::B => { 1.to_variant() },
            MyIntEnum::C => { 2.to_variant() },
        }
    }
}
```

At this point, there should be no problem in using `MyIntEnum` as a property in your native class that is exported to the editor.
