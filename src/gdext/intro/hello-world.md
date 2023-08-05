# Hello World

This page shows you how to develop your own small extension library and load it from Godot.
The tutorial is heavily inspired by [Creating your first script][tutorial-begin] from the official Godot documentation.
It is recommended to follow that alongside this tutorial, in case you're interested how certain GDScript concepts map to Rust.

## Table of contents
<!-- toc -->


## Godot project

Open the Godot project manager and create a blank Godot 4 project. Run the default scene to make sure everything is working.
Save the changes and consider versioning each step of the tutorial in Git.

Once you have a Godot project, you can register GDExtension libraries that it should use. These libraries are called _extensions_
and loaded during Godot engine startup -- both when opening the editor and when launching your game. The registration requires two files.

### `.gdextension` file

First, add `res://HelloWorld.gdextension`, which is the equivalent of `.gdnlib` for GDNative.

The `[configuration]` section should be copied as-is.  
The `[libraries]` section should be updated to match the paths of your dynamic Rust libraries.
`{myCrate}` can be replaced with the name of your crate.
```ini
[configuration]
entry_symbol = "gdext_rust_init"
compatibility_minimum = 4.1

[libraries]
linux.debug.x86_64 = "res://../rust/target/debug/lib{myCrate}.so"
linux.release.x86_64 = "res://../rust/target/release/lib{myCrate}.so"
windows.debug.x86_64 = "res://../rust/target/debug/{myCrate}.dll"
windows.release.x86_64 = "res://../rust/target/release/{myCrate}.dll"
macos.debug = "res://../rust/target/debug/lib{myCrate}.dylib"
macos.release = "res://../rust/target/release/lib{myCrate}.dylib"
macos.debug.arm64 = "res://../rust/target/debug/lib{myCrate}.dylib"
macos.release.arm64 = "res://../rust/target/release/lib{myCrate}.dylib"
```

```admonish note
For exporting your project, you'll need to use paths inside `res://`.
```

```admonish note
If you specify your cargo compilation target via the `--target` flag or a `.cargo/config.toml` file, the rust library will be placed in a path name that includes target architecture, and the `.gdextension` library paths will need to match. E.g. for M1 Macs (`macos.debug.arm64` and `macos.release.arm64`) the path would be `"res://../rust/target/aarch64-apple-darwin/debug/lib{myCrate}.dylib"`
```

### `extension_list.cfg`

A second file `res://.godot/extension_list.cfg` should be generated once you open the Godot editor for the first time.
If not, you can also manually create it, simply containing the Godot path to your `.gdextension` file:
```
res://HelloWorld.gdextension
```


## Cargo project

In your terminal, create a new Cargo project with your chosen crate name:
```bash
cargo new {myCrate} --lib
```

Then, open Cargo.toml and add the following:
```toml
[lib]
crate-type = ["cdylib"]

[dependencies]
godot = { git = "https://github.com/godot-rust/gdext", branch = "master" }
```

The `cdylib` crate type is a bit unusual in Rust. Instead of building a binary (`bin`, application) or library to be consumed by other Rust
code (`lib`), we create a _dynamic_ library exposing an interface in the _C_ programming language. This dynamic library is loaded by
Godot at runtime, through the GDExtension interface.

The main crate of gdext is called `godot`. At this point, it is still hosted on GitHub; in the future, it will be published to crates.io.
To fetch the latest changes, you can regularly run a `cargo update` (possibly breaking). Keep your `Cargo.lock` file under version control,
so that it's easy to revert updates.


## Rust entry point

Our C library needs to expose an _entry point_ to Godot: a C function that can be called through the GDExtension.
Setting this up requires quite some low-level FFI code, which is why gdext abstracts it for you.

In your `lib.rs`, add the following:
```rust
use godot::prelude::*;

struct MyExtension;

#[gdextension]
unsafe impl ExtensionLibrary for MyExtension {}
```

There are multiple things going on here:

1. Import the `prelude` module from the `godot` crate. This module contains the most common symbols in the gdext API.
2. Define a struct called `MyExtension`. This is just a type tag without data or methods, you can name it however you like.
3. Implement the `ExtensionLibrary` trait for our type, and mark it with the `#[gdextension]` attribute.

The last point declares the actual GDExtension entry point, and the proc-macro attribute takes care of the low-level details.


## Creating a Rust class

Now, let's write Rust code to define a _class_ that can be used in Godot.

Every class inherits an existing Godot-provided class (its _base class_ or just _base_).
Rust does not natively support inheritance, but the gdext API emulates it to a certain extent.


### Class declaration

In this example, we declare a class called `Player`, which inherits [`Sprite2D`][sprite2d-api] (a node type):
```rust
use godot::prelude::*;
use godot::engine::Sprite2D;

#[derive(GodotClass)]
#[class(base=Sprite2D)]
struct Player {
    speed: f64,
    angular_speed: f64,

    #[base]
    sprite: Base<Sprite2D>
}
```

Let's break this down.

1. The gdext prelude contains the most common symbols. Less frequent classes are located in the `engine` module.

2. The `#[derive]` attribute registers `Player` as a class in the Godot engine.
   See [API docs][derive-godotclass] for details about `#[derive(GodotClass)]`.

 ```admonish info
 `#[derive(GodotClass)]` _automatically_ registers the class -- you don't need an explicit 
 `add_class()` registration call, or a `.gdns` file as it was the case with GDNative.
 
 You will however need to restart the Godot editor to take effect.
 ```
   
3. The optional `#[class]` attribute configures how the class is registered. In this case, we specify that `Player` inherits Godot's
   `Sprite2D` class. If you don't specify the `base` key, the base class will implicitly be `RefCounted`, just as if you omitted the
   `extends` keyword in GDScript.

4. We define two fields `speed` and `angular_speed` for the logic. These are regular Rust fields, no magic involved. More about their use later.

5. The `#[base]` attribute declares the `sprite` field, which allows `self` to access the base instance (via composition, as Rust does not have
   native inheritance). 

   - The field must have type `Base<T>`. 
   - The name can be freely chosen. Here it's `sprite`, but `base` is also a common convention.
   - You do not _have to_ declare this field. If it is absent, you cannot access the base object from within `self`.
     This is often not a problem, e.g. in data bundles inheriting `RefCounted`.

```admonish warning
When adding an instance of your `Player` class to the scene, make sure to select node type `Player` and not its base `Sprite2D`.
Otherwise, your Rust logic will not run.

If Godot fails to load a Rust class (e.g. due to an error in your extension), it may silently replace it with its base class.
Use version control (git) to check for unwanted changes in `.tscn` files.
```


### Method declaration

Now let's add some logic. We start with overriding the `init` method, also known as the constructor. 
This corresponds to GDScript's `_init()` function.

```rust
use godot::engine::Sprite2DVirtual;

#[godot_api]
impl Sprite2DVirtual for Player {
    fn init(sprite: Base<Sprite2D>) -> Self {
        godot_print!("Hello, world!"); // Prints to the Godot console
        
        Self {
            speed: 400.0,
            angular_speed: std::f64::consts::PI,
            sprite
        }
    }
}
```
Again, those are multiple pieces working together, let's go through them one by one.

1. `#[godot_api]` - this lets gdext know that the following `impl` block is part of the Rust API to expose to Godot.
   This attribute is required here; accidentally forgetting it will cause a compile error.

2. `impl Sprite2DVirtual` - each of the engine classes has a `{ClassName}Virtual` trait, which comes with virtual functions for that
   specific class, as well as general-purpose functionality such as `init` (the constructor) or `to_string` (String conversion).
   The trait has no required methods.

3. The `init` constructor is an associated function ("static method" in other languages) that takes the base instance as argument and returns 
   a constructed instance of `Self`. While the base is usually just forwarded, the constructor is the place to initialize all your other fields.
   In this example, we assign initial values `400.0` and `PI`.

Now that initialization is sorted out, we can move on to actual logic. We would like to continuously rotate the sprite, and thus override
the `process()` method. This corresponds to GDScript's `_process()`. If you need a fixed framerate, use `physics_process()` instead.

```rust
use godot::engine::Sprite2DVirtual;

#[godot_api]
impl Sprite2DVirtual for Player {
    fn init(base: Base<Sprite2D>) -> Self { /* as before */ }

    fn physics_process(&mut self, delta: f64) {
        // In GDScript, this would be: 
        // rotation += angular_speed * delta
        
        self.sprite.rotate((self.angular_speed * delta) as f32);
        // The 'rotate' method requires a f32, 
        // therefore we convert 'self.angular_speed * delta' which is a f64 to a f32
    }
}
```

GDScript uses property syntax here; Rust requires explicit method calls instead. Also, access to base class methods -- such as `rotate()` in
this example -- is done via the `#[base]` field.

This is a point where you can compile your code, launch Godot and see the result. The sprite should rotate at a constant speed.

![rotating sprite](https://docs.godotengine.org/en/stable/_images/scripting_first_script_godot_turning_in_place.gif)

```admonish tip
 **Launching the Godot application**
 
While it's possible to open the Godot editor and press the launch button every time you made a change in Rust, this is not the most
efficient workflow. Unfortunately there is [a GDExtension limitation][no-reload] that prevents recompilation while the editor is open 
 (at least on Windows systems -- it tends to work better on Linux and macOS).
 
 However, if you don't need to modify anything in the editor itself, you can launch Godot from the command line or even your IDE.
 Check out the [command-line tutorial][godot-command-line] for more information.
```

We now add a translation component to the sprite, following [the upstream tutorial][tutorial-full-script].

```rust
use godot::engine::Sprite2DVirtual;

#[godot_api]
impl Sprite2DVirtual for Player {
    fn init(base: Base<Sprite2D>) -> Self { /* as before */ }

    fn physics_process(&mut self, delta: f64) {
        // GDScript code:
        //
        // rotation += angular_speed * delta
        // var velocity = Vector2.UP.rotated(rotation) * speed
        // position += velocity * delta
        
        self.sprite.rotate((self.angular_speed * delta) as f32);

        let rotation = self.sprite.get_rotation();
        let velocity = Vector2::UP.rotated(rotation) * self.speed as f32;
        self.sprite.translate(velocity * delta as f32);
        
        // or verbose: 
        // self.sprite.set_position(
        //     self.sprite.get_position() + velocity * delta as f32
        // );
    }
}
```

The result should be a sprite that rotates with an offset.

![rotating translated sprite](https://docs.godotengine.org/en/stable/_images/scripting_first_script_rotating_godot.gif)

### Custom Rust APIs

Say you want to add some functionality to your `Player` class, which can be called from GDScript. For this, you have a separate `impl` block,
again annotated with `#[godot_api]`. However, this time we are using an _inherent_ `impl` (i.e. without a trait name).

Concretely, we add a function to increase the speed, and a signal to notify other objects of the speed change.
```rust
#[godot_api]
impl Player {
	#[func]
	fn increase_speed(&mut self, amount: f64) {
		self.speed += amount;
		self.emit_signal("speed_increased".into(), &[]);
	}

	#[signal]
	fn speed_increased();
}
```
`#[godot_api]` takes again the role of exposing the API to the Godot engine. But there are also two new attributes:

- `#[func]` exposes a function to Godot. The parameters and return types are mapped to their corresponding GDScript types. 
- `#[signal]` declares a signal. A signal can be emitted with the `emit_signal` method (which every Godot class provides, since it is inherited
  from `Object`).

API attributes typically follow the GDScript keyword names: `class`, `func`, `signal`, `export`, `var`, ...

That's it for the _Hello World_ tutorial! The following chapters will go into more detail about the various features that gdext provides.


[derive-godotclass]: https://godot-rust.github.io/docs/gdext/master/godot/prelude/derive.GodotClass.html
[godot-command-line]: https://docs.godotengine.org/en/stable/tutorials/editor/command_line_tutorial.html
[no-reload]: https://github.com/godotengine/godot/issues/66231
[sprite2d-api]: https://godot-rust.github.io/docs/gdext/master/godot/engine/struct.Sprite2D.html
[tutorial-begin]: https://docs.godotengine.org/en/stable/getting_started/step_by_step/scripting_first_script.html
[tutorial-full-script]: https://docs.godotengine.org/en/stable/getting_started/step_by_step/scripting_first_script.html#complete-script
