# Setup

Before we can start creating a hello-world project using godot-rust, we'll need to install the necessary software.

## Godot Engine

The default API version is currently 4. For the rest of the tutorial, we'll assume that you have Godot 4 installed, and available in your `PATH` as `godot4` or a system variable called `GODOT4_BIN`.

You may download binaries of Godot 4 from the official repository: [https://godotengine.org/download/](https://godotengine.org/download/).

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

When working on Windows, it's also necessary to install the [Visual Studio Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/), or the full Visual Studio (*not* Visual Studio Code). More details can be found on [Working with Rust on Windows](https://github.com/rust-lang/rustup#working-with-rust-on-windows). Note that LLVM is also required on top of those dependencies to build your godot-rust project, see the next section for more information.

## LLVM

The godot-rust bindings depend on `bindgen`, which in turn [depends on LLVM](https://rust-lang.github.io/rust-bindgen/requirements.html). You may download LLVM binaries from [https://releases.llvm.org/](https://releases.llvm.org/).

After installation, check that LLVM was installed properly:

```bash
# Check if Clang is installed and registered in PATH
clang -v
```