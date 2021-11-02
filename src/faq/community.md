# FAQ: Community

## Table of contents
<!-- toc -->

## I need help, where can I ask?

The godot-rust project uses several different sources for different kinds of information.

### 1 - The godot-rust documentation is hosted at [docs.rs](https://docs.rs/gdnative/)

Specific places to start in the  would be the following:
- [API documentation](https://docs.rs/gdnative/latest/gdnative/api/index.html) this is useful to find the necessary includes from the Godot API that are not directly implemented in the preludes.
  - [Godot Object](https://docs.rs/gdnative/latest/gdnative/trait.GodotObject.html) includes all of the information about the trait that all Godot API objects implement. 
  - [NativeClass](https://docs.rs/gdnative/latest/gdnative/derive.NativeClass.html) includes all the derive macro attributes that you can use to integrate your script class with godot.

### 2 - The Official Godot Docs

As godot-rust is the layer that wraps the [Godot API](https://docs.godotengine.org/en/stable/classes/index.html), the effects and use cases for many of the methods are available in the Official Docs. Consulting this is a great place whenever you have questions for anything specific to Godot that is outside the bounds of godot-rust.

[Godot docs](https://docs.godotengine.org/en/stable/index.html)

### 3 - The Godot Rust Discord

For more complex questions the fastest way to get an answer is to reach out to us and fellow developers on the [godot-rust Discord](https://discord.gg/FNudpBD).

### 4 - If it appears to be a bug, open a Github issue

If you find a bug or an issue with the godot-rust bindings or have an idea for future development, please feel free to open up a Github issue and we'd be happy to take a look.

Please note, due to the limited resources for maintaining this project, bugs generally take precendence over feature requests.

In addition, Github issues are not intended for general questions as only project maintainers and contributors generally read and track new issues. As such, we would highly recommend seeking out one of the above options before opening an issue if you have a question.