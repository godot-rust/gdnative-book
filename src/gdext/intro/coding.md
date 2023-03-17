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
To add Rust functionality to the node, we need to use a base class which we can build on. In this case, the base is `CharacterController2D`. To use it, we must "inherit" it. Rust doesn't natively support inheritance, but gdext has a way to emulate the Godot inheritance hierarchy: by using the `#[class]` attribute. Like this:
```rust
use godot::engine::CharacterController2D;
use godot::prelude::*;

#[derive(GodotClass)]
#[class(base=CharacterController2D)]
struct Player {
    speed: f32,
    gravity: f32,

    #[base]
    base: Base<CharacterController2D>
}
```
Let's break this down. As mentioned early, we need the `#[class]` attribute, but what does it do? The attribute allows use to store the inherited functions and values in a field we can use to interact with Godot. The derive allows gdext to use the struct and turn it into a Godot class. The `#[class]` attribute allows you to customize the node. In this case we made the base `CharacterController2D`. The `speed` and `gravity` fields used for a the player. You don't need to add them to make a node, but in this case we want them.

Now let's add some functionality.
```rust
impl GodotExt for Player {
    fn init(base: Base<CharacterController2D>) -> Self {
        Self {
            speed: 5.0,
            gravity: 5.0,
            base
        }
    }

    fn physics_process(&mut self, delta: f64) {
        if Engine::singleton().is_hint_editor() {
            return
        }

        self.base.set_velocity(Vector2::new(speed, gravity));
        self.base.move_and_slide();
    }
}
```
`GodotExt` is a trait that holds functions for Godot to use. We are defining 2 functions `init` and `physics_process`. `init` creates a new node for Godot. It holds a simple Self struct and returns Player. `physics_process` runs physics logic every frame. This holds the parameter `delta`. `delta` is the time between the current frame and the last frame. The if statement checks if it is running in the editor or in the actually game. This is because or code will start running in the engine. `self.base.set_velocity(Vector::new(speed, gravity));` tells the engine that the new velocity of the node is the vector 2(in this case it's speed on the x axis and gravity on the y axis). `self.base.move_and_slide();` takes the velocity and moves it while allowing for collsions. 