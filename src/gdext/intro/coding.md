# Coding
This is where you can write the Rust logic.

## Creating the project
In your terminal run:
```bash
cargo new name --lib
```
Then open the Cargo.toml and write the following:
```toml
[lib]
crate-type = ["cdylib"]

[dependencies]
godot = { git = "https://github.com/godot-rust/gdext", branch = "master" }
```

Now we need to add the entry symbol. In the lib.rs file write:
```rust
use godot::prelude::*;

struct Main;

#[gdextension]
unsafe impl ExtensionLibrary for Main {}
```

## Creating a Node Type
To add Rust functionality to the node, we need to use a base class which we can build on. In this case, the base is `CharacterBody2D`. To use it, we must "inherit" it. Rust doesn't natively support inheritance, but gdext has a way to emulate the Godot inheritance hierarchy: by using the `#[class]` attribute with the `GodotClass` derive macro. Like this:
```rust
use godot::engine::{Engine, CharacterBody2D, CharacterBody2DVirtual};
use godot::prelude::*;

#[derive(GodotClass)]
#[class(base=CharacterBody2D)]
struct Player {
    speed: f32,
    gravity: f32,

    #[base]
    base: Base<CharacterBody2D>
}
```
Let's break this down. As mentioned early, we need the `#[class]` attribute, but what does it do? The attribute allows use to store the inherited functions and values in a field we can use to interact with Godot. The derive allows gdext to use the struct and turn it into a Godot class. The `#[class]` attribute allows you to customize the node. In this case we made the base `CharacterBody2D`. The `speed` and `gravity` fields used for a the player. You don't need to add them to make a node, but in this case we want them.

Now let's add some functionality.
```rust
#[godot_api]
impl CharacterBody2DVirtual for Player {
    fn init(base: Base<CharacterBody2D>) -> Self {
        Self {
            speed: 5.0,
            gravity: 5.0,
            base
        }
    }

    fn physics_process(&mut self, delta: f64) {
        // Note: classes will run inside of Godot itself, potentially leading to error spam - see issue #70
        // This prevents our function from running if it's being run inside the editor.
        if Engine::singleton().is_editor_hint() {
            return
        }

        let velocity = Vector2::new(self.speed, self.gravity);
        self.set_velocity(velocity);
        self.move_and_slide();
    }
}
```
There are two important pieces working together here:
1. `#[godot_api]` - This attribute handles properly delegating the virtual methods being implemented. This attribute is necessary on every impl with functions that need to interact with Godot. If you forget to add it, you will see an error message saying the `You_forgot_the_attribute__godot_api` trait has not been implemented, which is also handled by `#[godot_api]`.
2. `impl CharacterBody2DVirtual` - Each of the engine classes has a `{ClassName}Virtual` trait that you can inherit from to override the constructor and virtual functions. Notably, if you do not implement the `init` function (or use the `#[class(init)]` syntax when defining the struct - see the docs on the `GodotClass` derive macro), Godot will not be able to initialize your type.

We are defining 2 functions `init` and `physics_process`. `init` creates a new node for Godot. It holds a simple Self struct and returns Player. `physics_process` runs physics logic every frame. This holds the parameter `delta`. `delta` is the time between the current frame and the last frame. The if statement checks if it is running in the editor or in the actually game. This is because or code will start running in the engine. `self.set_velocity(velocity);` tells the engine that the new velocity of the node is the vector 2(in this case it's speed on the x axis and gravity on the y axis). `self.move_and_slide();` takes the velocity and moves it while allowing for collsions.

### Inherent Impls
Say you want to add some functionality unique to your Player in an inherent impl, with some function and signals callable by Godot:
```rust
#[godot_api]
impl Player {
	#[func]
	fn increase_speed(&mut self) {
		self.speed += 1.0;
		self.emit_signal("speed_increased".into(), &[]);
	}

	#[signal]
	fn speed_increased();
}
```
Again we see the `#[godot_api]` macro doing some more heavy lifting. But there are also two new attributes:
- `#[func]` - exposes a function to Godot. Allows you to call it from GDScript, connect it to a signal, etc.
- `#[signal]` - Declares a signal. A signal can be emitted with the `emit_signal` method of any `Object` subclass. (Note - as of 2023-04-09, signals do not allow for parameters)

## The `Gd` smart pointer
Any time you receive an object from a Godot API (i.e. `get_node_as`) it will be a `Gd<T>`. More information on this type can be found in the crate's docs. For example, if you were writing a signal handler for the `body_entered` signal of a `Node3D`, it might look like this:
```rust
#[func]
fn on_body_entered(&mut self, body: Gd<Node3D>) {
	// ...
}
```
