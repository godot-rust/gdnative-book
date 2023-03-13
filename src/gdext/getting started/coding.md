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
To control a node you need to create a node type that inherits the node type. For an example, Ley