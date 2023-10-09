# Data representations

The godot-rust library uses many different approaches to store and transport data. This chapter explains high-level concepts of related terminology used throughout the library and its documentation. It is not a usage guide however -- to see the concepts in action, check out [Binding to Rust code](../rust-binding.md).


## Object and class

Godot is built around _classes_, object-oriented types in a hierarchy, with the base class `Object` at the top. When talking about classes, we explicitly mean classes in the `Object` hierarchy and not built-in types like `String`, `Vector2`, `Color`, even though they are technically classes in C++. In Rust, classes are represented as structs.

Every user-defined class inherits `Object` directly or indirectly, and thus all methods defined in `Object` are accessible on _any_ instance of a user-defined class. This type includes functionality for:
* object lifetime: `_init` (`new` in Rust), `free`
* identification and printing: `to_string`, `get_instance_id`
* reflection/introspection: `get_class`, `get`, `has_method`, ...
* custom function invocation: `call`, `callv`, `call_deferred`
* signal handling: `connect`, `emit_signal`, ...

`Object` itself comes with manual memory management. All instances must be deallocated using the `free()` method. This is typically not what you want, instead you will most often work with the following classes inherited from `Object`:

* **`Reference`**  
  Reference-counted objects. This is the default base class if you don't use the `extends` keyword in GDScript. Allows to pass around instances of this type freely, managing memory automatically when the last reference goes out of scope.  
  Do not confuse this type with the godot-rust `Ref` smart pointer.
* **`Node`**  
  Anything that's part of the scene tree, such as `Spatial` (3D), `CanvasItem` and `Node2D` (2D). Each node in the tree is responsible of its children and will deallocate them automatically when it is removed from the tree. At the latest, the entire tree will be destroyed when ending the application.  
  **Important:** as long as a node is not attached to the scene tree, it behaves like an `Object` instance and must be freed manually. On the other hand, as long as it is part of the tree, it can be destroyed (e.g. when its parent is removed) and other references pointing to it become invalid.
* **`Resource`**  
  Data set that is loaded from disk and cached in memory, for example 3D meshes, materials, textures, fonts or music (see also [Godot tutorial](https://docs.godotengine.org/en/stable/getting_started/step_by_step/resources.html)).
  `Resource` inherits `Reference`, so in the context of godot-rust, it can be treated like a normal, reference-counted class. 

When talking about inheritance, we always mean the relationship in GDScript code. Rust does not have inheritance, instead godot-rust implements `Deref` traits to allow implicit upcasts. This enables to invoke all parent methods and makes the godot-rust API very close to GDScript.

Classes need to be added as `NativeScript` resources inside the Godot editor, see [here](../intro/hello-world.html#creating-the-nativescript-resource) for a description.

_See `Object` in
[godot-rust docs](https://docs.rs/gdnative/latest/gdnative/api/struct.Object.html),
[Godot docs](https://docs.godotengine.org/en/latest/classes/class_object.html)_  
_See `GodotObject`, the Rust trait implemented for all Godot classes, in [godot-rust docs](https://docs.rs/gdnative/latest/gdnative/trait.GodotObject.html)_


## Variant

`Variant` is a type that can hold an instance of _any_ type in Godot. This includes all classes (of type `Object`) as well as all built-in types such as `int`, `String`, `Vector2` etc.

Since GDScript is a dynamic language, you often deal with variants implicitly. Variables which are not type-annotated can have values of multiple types throughout their lifetime. In static languages like Rust, every value must have a defined type, thus untyped values in GDScript correspond to `Variant` in Rust. Godot APIs which accept any type as parameter are declared as `Variant` in the GDNative bindings (and thus godot-rust library). Sometimes, godot-rust also provides transparent mapping from/to concrete types behind the scenes.

Variants also have a second role as a serialization format between Godot and Rust. It is possible to extend this beyond the built-in Godot types. To make your own types convertible from and to variants, implement the traits [`FromVariant`](https://docs.rs/gdnative/latest/gdnative/core_types/trait.FromVariant.html) and [`ToVariant`](https://docs.rs/gdnative/latest/gdnative/core_types/trait.ToVariant.html). Types that can only be safely converted to variants by giving up ownership can use [OwnedToVariant](https://docs.rs/gdnative/latest/gdnative/core_types/trait.OwnedToVariant.html), which is similar to the Rust `Into` trait.

_See `Variant` in
[godot-rust docs](https://docs.rs/gdnative/latest/gdnative/core_types/struct.Variant.html),
[Godot docs](https://docs.godotengine.org/en/latest/classes/class_variant.html)_


## Script

Scripts are programmable building blocks that can be attached to nodes in the scene tree, in order to customize their behavior. Depending on the language in which the script is written, there are different classes which inherit the `Script` class; relevant here will be `NativeScript` for classes defined in Rust, and `GDScript` for classes defined in GDScript. Scripts are stored as Godot resources (like materials, textures, shaders etc), usually in their own separate file.

Scripts _always_ inherit another class from Godot's `Object` hierarchy, either an existing one from Godot or a user-defined one. In Rust, scripts are limited to inherit an existing Godot class; other scripts cannot be inherited. This makes each script a class on their own: they provide the properties and methods from their _base object_, plus all the properties and methods that you define in the script. 

_See `Script` in
[godot-rust docs](https://docs.rs/gdnative/latest/gdnative/api/struct.Script.html),
[Godot docs](https://docs.godotengine.org/en/latest/classes/class_script.html)_
