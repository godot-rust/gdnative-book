# Setup

Before we can start creating a hello-world project using godot-rust, we'll need to install the necessary software.

## Godot Engine

The default API version is currently 3.2.3-stable. For the rest of the tutorial, we'll assume that you have Godot 3.2.3-stable installed, and available in your `PATH` as `godot`.

You may download binaries of Godot 3.2.3-stable from the official repository: [https://downloads.tuxfamily.org/godotengine/3.2.3/](https://downloads.tuxfamily.org/godotengine/3.2.3/).

> ### Using another build of the engine
>
> For simplicity, we assume that you use the official build of 3.2.3-stable for the Getting Started tutorial. If you want to use another version of the engine, see the [Using custom builds of Godot](../advanced-guides/custom-godot.md) guide.

## Rust

[rustup](https://rustup.rs/) is the recommended way to install the Rust toolchain, including the compiler, standard library, and Cargo, the package manager. Visit [https://rustup.rs/](https://rustup.rs/) to see instructions for your platform.

After installation of rustup and the `stable` toolchain, check that they were installed properly:

```bash
# Check Rust toolchain installer version
rustup -V
# Check Rust version
rustc --version
# Check Cargo version
cargo -V
```

### Windows

When working on Windows, it's also necessary to install the [Visual Studio Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/), or the full Visual Studio (*not* Visual Studio Code). More details can be found on [Working with Rust on Windows](https://github.com/rust-lang/rustup#working-with-rust-on-windows).

## LLVM

The godot-rust bindings depend on `bindgen`, which in turn [depends on LLVM](https://rust-lang.github.io/rust-bindgen/requirements.html). You may download LLVM binaries from [https://releases.llvm.org/](https://releases.llvm.org/).

After installation, check that LLVM was installed properly:

```bash
# Check if Clang is installed and registered in PATH
clang -v
```

`bindgen` may complain about a missing `llvm-config` binary, but it is not actually required to build the `gdnative` crate. If you see a warning about `llvm-config` and a failed build, it's likely that you're having a different problem!


## Using the template

We also provide an experimental [template](https://github.com/godot-rust/godot-rust-template) to get you started right away. All the boilerplate stuff is already done for you. If you have decided to use the template, you can skip section 2.2. Check out the [wiki](https://github.com/godot-rust/godot-rust-template/wiki) for instructions.

The template is currently a work-in-progress, and might not work in all setups where the base library would be compatible. If you encounter any issues with the template, please report them at its [issue tracker](https://github.com/godot-rust/godot-rust-template/issues/).

### Setup
```shell
$ cargo install cargo-generate cargo-make
```

### Usage
```shell
$ cargo generate --git https://github.com/godot-rust/godot-rust-template --name my-awesome-game
$ cd my-awesome-game
$ cargo make run
```
