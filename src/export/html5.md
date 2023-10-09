# HTML5

Exporting to HTML5 works just like exporting to other platforms, however there are some things that can make the process a bit tricky and require extra attention.

## What you need (to know)

The Godot HTML5 export templates are built with [Emscripten](https://emscripten.org/), so you need to build your Rust code for the `wasm32-unknown-emscripten` target (`wasm32-unknown-unknown` will not work). Since Emscripten does not offer a stable ABI between versions, your code and the Godot export template have to be built with the same Emscripten version, which also needs to be compatible with the Rust compiler version you are using.

In practice, this means you probably have to do some or all of these things:

* install a specific version of Emscripten
* [build the HTML5 export template yourself](https://docs.godotengine.org/en/stable/development/compiling/compiling_for_web.html) 
* use a specific version of Rust

You can check which versions of Emscripten/Rust you are using like this:

```bash
emcc -v
rustc --version --verbose
```

A list of compatible Rust and Emscripten versions are given in the next section.

Emscripten uses a cache to store build artifacts and header files. This means that the order in which steps are completed matters. In particular, Godot export template should be built before the GDNative library.

Also note that `wasm32-unknown-emscripten` is a 32-bit target, which can cause problems with crates incorrectly assuming that `usize` is 64 bits wide.

### Limitations

* Multi-threading is not supported. See [this issue](https://github.com/godot-rust/gdnative/issues/1022) for more details and instructions for failing experimental build.
* Debug builds may not work due to their large size. See [the corresponding issue](https://github.com/godot-rust/gdnative/issues/1021).
  * As a workaround, you can set `opt-level = 1` in debug builds, which reduces the build size. Add the following to your workspace's/project's `Cargo.toml`:
```toml
[profile.dev]
opt-level = 1
```

## Godot 3.5

**Disclaimer**: _Currently, the following steps are only tested and confirmed to work on Linux._

[Godot 3.5's prebuilt HTML5 export template is built with Emscripten 3.1.10](https://github.com/godotengine/godot/blob/3.5/.github/workflows/javascript_builds.yml). It might be possible to use it if build your Rust code with that exact version, but extra compiler flags may be needed. This guide focuses on building the export template yourself with a recent version of Emscripten.

As of 2023-01-28 you also need to use a Rust nightly build.

Confirmed working versions are:

* Rust toolchain `nightly-2023-01-27` and `Emscripten 3.1.21 (f9e81472f1ce541eddece1d27c3632e148e9031a)`
  * This combination will be used in the following sections of this tutorial.
* `rustc 1.65.0-nightly (8c6ce6b91 2022-09-02)` and `Emscripten 3.1.21-git (3ce1f726b449fd1b90803a6150a881fc8dc711da)`

The compatibility problems between Rust and Emscripten versions are assumed to stem from differing LLVM versions. You can use the following commands to check the LLVM versions of the currently installed nightly toolchain and the currently active emcc:

```bash
rustc +nightly --version --verbose
emcc -v
```

`emcc -v` reports the version number of `clang`, which is equal to the version number of the overall LLVM release.

### Install Rust toolchain

Following bash commands install `nightly-2023-01-27` toolchain with `wasm32-unknown-emscripten` target:

```bash
# Install the specific version of Rust toolchain.
rustup toolchain install nightly-2023-01-27

# Add wasm32-unknown-emscripten target to this toolchain.
rustup +nightly-2023-01-27 target add wasm32-unknown-emscripten
```

### Installing and configuring Emscripten

Use the following bash commands to install Emscripten 3.1.21, which is known to be compatible with Rust `nightly-2023-01-27`:

```bash
# Get the emsdk repo.
git clone https://github.com/emscripten-core/emsdk.git

# Enter the cloned directory.
cd emsdk

# Download and install a compatible SDK version.
./emsdk install 3.1.21

# Make this SDK version "active" for the current user. (writes .emscripten file)
./emsdk activate 3.1.21

# Activate PATH and other environment variables in the current terminal
source ./emsdk_env.sh
```

### Building Godot

Build the Godot Export template according to the [instructions](https://docs.godotengine.org/en/stable/development/compiling/compiling_for_web.html):

```bash
source "/path/to/emsdk-portable/emsdk_env.sh"
scons platform=javascript tools=no gdnative_enabled=yes target=release
mv bin/godot.javascript.opt.gdnative.zip bin/webassembly_gdnative_release.zip
```

Since this is a web build, you might want to disable unused modules and optimize your build for size [as shown in Godot documentation](https://docs.godotengine.org/en/stable/development/compiling/optimizing_for_size.html).

Set the newly built export template as a [custom template](https://user-images.githubusercontent.com/2171264/175822720-bcd2f1ff-0a1d-4495-9f9c-892d42e9bdcd.png) in Godot and be sure to set the export type as GDNative. When exporting, uncheck "Export With Debug".

### Building your Rust code

In your project's `.cargo/config.toml` [(see Cargo Docs)](https://doc.rust-lang.org/cargo/reference/config.html), add the following:

```toml
[target.wasm32-unknown-emscripten]
rustflags = [
	"-Clink-arg=-sSIDE_MODULE=2", # build a side module that Godot can load
	"-Zlink-native-libraries=no", # workaround for a wasm-ld error during linking
	"-Cpanic=abort", # workaround for a runtime error related to dyncalls
]
```

Build like this:

```bash
source "/path/to/emsdk-portable/emsdk_env.sh"
export C_INCLUDE_PATH=$EMSDK/upstream/emscripten/cache/sysroot/include
	
cargo +nightly-2023-01-27 build --target=wasm32-unknown-emscripten --release
```

This will produce a `.wasm` file in `target/wasm32-unknown-emscripten/release/` directory. Add this file to `GDNativeLibrary` properties under `entry/HTML5.wasm32`, just like you would add a `.so` or a `.dll` for Linux or Windows.

### Errors you might encounter

Compile time:
* `failed to run custom build command for gdnative-sys v0.10.1`, `fatal error: 'wchar.h' file not found`: Emscripten cache not populated, build Godot export template first
* `undefined symbol: __cxa_is_pointer_type`: You need to build with `-Clink-arg=-sSIDE_MODULE=2`
* `error: undefined symbol: main/__main_argc_argv (referenced by top-level compiled C/C++ 
code)`: Your `.cargo/config.toml` was not loaded. See [Cargo Docs](https://doc.rust-lang.org/cargo/reference/config.html) and verify its location.

Runtime:
* `indirect call signature mismatch`: Possibly due to Emscripten version mismatch between Godot and Rust
* `need the dylink section to be first`: Same as above
* `WebAssembly.Module(): Compiling function #1 failed: invalid local index`:  You need to build with `-Cpanic=abort`
* `ERROR: Can't resolve symbol godot_gdnative_init. Error: Tried to lookup unknown symbol "godot_gdnative_init" in dynamic lib`: Possibly because Emscripten version is not compatible with Rust toolchain version. See the compatible version list above.
* `wasm validation error: at offset 838450: too many locals`: Binary size is too big. See [limitations](#limitations) section for a workaround.
* `WebAssembly.instantiate(): Compiling function #2262:"gdnative_sys::GodotApi::from_raw::..." failed: local count too large`: Binary size is too big. See [limitations](#limitations) section for a workaround.

### Further reading

In the future, some of this will probably not be needed, for more info see:
* [Tracking issue for godot-rust wasm support](https://github.com/godot-rust/godot-rust/issues/647)
* [Rust](https://github.com/rust-lang/rust/issues/98155) and [Emscripten](https://github.com/rust-lang/rust/pull/98303#issuecomment-1162172132) issues that explain the need for `-Zlink-native-libraries=no`
* [Emscripten PR that should have obsoleted `-Cpanic=abort`](https://github.com/emscripten-core/emscripten/pull/17328)
* [Rust PR that would obsolete `-Clink-arg=-sSIDE_MODULE=2`](https://github.com/rust-lang/rust/pull/98358)
