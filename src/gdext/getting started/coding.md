# Coding
This is where you can write the rust logic.

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
To control a node you need to create a node type that inherits the node type. For an example, Let's use a CharacterController2D.
In a new file, create a struct. Like this:
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
Let's break this down. The derive allows gdextensions to use the struct and turn it into a godot class. The class attribute allows you customize the Node. In this case we made the base Character Controller 2D. Then, in the struct speed and gravity fields are not important just help with player speed. The base field lets edextensions know where to store the Node data. 

Now let's add some functionality.
```rust
impl GodotExt for Player {
    fn init(base: Base<CharacterController2D>) -> Self {
        Self {
            speed: 5f32,
            gravity: 5f32,

            base
        }
    }

    fn physics_process(&mut self, delta: f64) {
        if Engine::singleton().is_hint_editor() {
            return
        }

        self.base.set_velocity(Vector2::new(speed, gravity));
        self.base.move_slide();
    }
}
```
This code starts by setting some functions that godot will run. The init function is for Godot to create our Node and the physics_process function is what godot runs every frame. The if statement checks if the game is running and doesn't run in the editor. The set_velocity setst eh velocity of the node and the move and slide moves with collisions.