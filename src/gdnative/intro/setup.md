# Setup

Before we can start creating a hello-world project using godot-rust, we'll need to install the necessary software.

## Godot Engine

The default API version is currently 3.2.3-stable. For the rest of the tutorial, we'll assume that you have Godot 3.2.3-stable installed, and available in your `PATH` as `godot`.

You may download binaries of Godot 3.2.3-stable from the official repository: [https://downloads.tuxfamily.org/godotengine/3.2.3/](https://downloads.tuxfamily.org/godotengine/3.2.3/).

> ### Using another build of the engine
>
> For simplicity, we assume that you use the official build of 3.2.3-stable for the Getting Started tutorial. If you want to use another version of the engine, see the [Using custom builds of Godot](../advanced/custom-godot.md) guide.

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

## Windows

When working on Windows, it's also necessary to install the [Visual Studio Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/), or the full Visual Studio (*not* Visual Studio Code). More details can be found on [Working with Rust on Windows](https://github.com/rust-lang/rustup#working-with-rust-on-windows). Also note that LLVM is also required on top of those dependencies to be able to build your godot-rust project, see the next section for more information.

## LLVM

The godot-rust bindings depend on `bindgen`, which in turn [depends on LLVM](https://rust-lang.github.io/rust-bindgen/requirements.html). You may download LLVM binaries from [https://releases.llvm.org/](https://releases.llvm.org/).

After installation, check that LLVM was installed properly:

```bash
# Check if Clang is installed and registered in PATH
clang -v
```

`bindgen` may complain about a missing `llvm-config` binary, but it is not actually required to build the `gdnative` crate. If you see a warning about `llvm-config` and a failed build, it's likely that you're having a different problem!


## Using the template

One way to get started with godot-rust is a full-fledged (inofficial) template, which can be found [here](https://github.com/macalimlim/godot-rust-template) to get you started right away. All the boilerplate stuff is already done for you, however, using the template requires you to set up extra dependencies and toolchains. Check out the [wiki](https://github.com/macalimlim/godot-rust-template/wiki) for instructions on how to get started with the template.

The template is not maintained by us, and might not work in all setups where the base library would be compatible. If you encounter any issues with the template, please report them at its [issue tracker](https://github.com/macalimlim/godot-rust-template/issues/).
