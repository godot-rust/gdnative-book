# Projects: Tools and integrations

This page lists projects, which are intended to be used as an extension to godot-rust.  
Examples include:

* CLI or other tools enhancing the development workflow
* Libraries that directly enhance the godot-rust experience
* Libraries that connect godot-rust with other crates in the Rust ecosystem

This page should only provide a high-level description of each project (a couple sentences), plus relevant links and optionally one screenshot. It should _not_ include tutorials or extended code examples, as they tend to become outdated very quickly. Instead, the project's repository or homepage is a much better place for advertising the concrete functionality the tool offers and providing introductory examples.


### Table of contents
<!-- toc -->

## godot-egui

_[GitHub](https://github.com/setzer22/godot-egui)_

![](img/godot-egui.gif)

**godot-egui** is an [egui](https://github.com/emilk/egui) backend for godot-rust. 

This crate enables using the immediate-mode UI library egui within Godot applications and games. Updating UI widgets and properties directly from Rust code becomes straightforward, without going through Godot nodes, signals and the intricacies of GDNative's ownership semantics.


## godot-rust-cli

_[GitHub](https://github.com/robertcorponoi/godot-rust-cli)_

**Godot Rust CLI** is a simple command-line interface to help you create and update Rust components for your Godot projects.

Example:
```sh
# create a Rust library `rust_modules` for the `platformer` Godot project
godot-rust-cli new rust_modules platformer

# create player.rs + player.gdns
godot-rust-cli create Player 

# generate dynamic library to be called by Godot, automatically watch changes
godot-rust-cli build --watch
```

Note that there is also [godot-rust-cli-upgrader](https://github.com/robertcorponoi/godot-rust-cli-upgrader) to upgrade the CLI.


## ftw

_[GitHub](https://github.com/macalimlim/ftw)_

**ftw** is a command-line interface to manage your godot-rust project. It enables you to set up projects, add native classes which are automatically wired up, and run build commands.

Example:

```sh
# create new project using the default template
ftw new my-awesome-game

# create a class that derives from `Area2D`
ftw class MyHero Area2D

# create a class called `MySingleton` that derives from `Node`
ftw singleton MySingleton
 
# build the library for the `linux-x86_64` platform using `debug` as default
ftw build linux-x86_64
```


## gdrust

_[GitHub](https://github.com/wyattjsmith1/gdrust)_

**gdrust** is a an extension library to godot-rust. It adds ergonomic improvements and is an inspiration for godot-rust itself.

Example:

```rs
#[gdrust]
#[signal(signal_name(arg_name: I64, arg2_name: F64 = 10.0))]
struct MyClass {
    #[export_range(1, 10, 2, "or_greater")]
    my_range: i32,
}
```

