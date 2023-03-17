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
To add Rust functionality to the node, we need to use a base class which we can build on. In this case, the base is `CharacterController2D`. To use it we must inherit it. Rust doesn't natively support inheritance, but gdext has a way to emulate the Godot inheritance hierarchy: by using the `class` attribute. 
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
Let's break this down. As mentioned early, we need the `#[class]` attribute, but what does it do? The attribute allows use to store the inherited functions and values in a field we can use to interact with Godot. The derive allows gdext to use the struct and turn it into a Godot `#[class]`. The `#[class]` attribute allows you to customize the node. In this case we made the base `CharacterController2D`. The `speed` and `gravity` fields used for a the player. You don't need to add them to make a node, but in this case we want them.

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
This code starts by setting some functions that godot will run. The init function is for Godot to create our Node and the physics_process function is what godot runs every frame. The if statement checks if the game is running and doesn't run in the editor. The set_velocity setst eh velocity of the node and the move and slide moves with collisions.