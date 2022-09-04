# HTML5

Exporting to HTML5 works just like exporting to other platforms, however there are some things that can make the process a bit tricky and require extra attention.

## What you need (to know)

The Godot HTML5 export templates are built with [Emscripten](https://emscripten.org/), so you need to build your Rust code for the `wasm32-unknown-emscripten` target (`wasm32-unknown-unknown` will not work).  
Since Emscripten does not offer a stable ABI between versions, your code and the Godot export template have to be built with the same Emscripten version, which also needs to be compatible with the Rust compiler version you are using.
In practice, this means you probably have to do some or all of these things:
* install a specific version of Emscripten
* [build the HTML5 export template yourself](https://docs.godotengine.org/en/stable/development/compiling/compiling_for_web.html) 
* use a specific version of Rust

You can check which versions of Emscripten/Rust you are using like this:

```bash
emcc -v
rustc --version --verbose
```

A version mismatch between Rust and Emscripten will probably result in compiler errors, while using incompatible Emscripten versions between Rust and Godot will result in cryptic runtime errors.

Emscripten uses a cache to store build artifacts and header files. In some cases this means that the order in which steps are completed matters.

Also note that `wasm32-unknown-emscripten` is a 32-bit target, which can cause problems with crates incorrectly assuming that `usize` is 64 bits wide.

## Godot 3.5

**Disclaimer**: _Currently, the following steps are only tested and confirmed to work on Linux._

[Godot 3.5's prebuilt HTML5 export template is built with Emscripten 3.1.10](https://github.com/godotengine/godot/blob/3.5/.github/workflows/javascript_builds.yml).
It might be possible to use it if build your Rust code with that exact version, but extra compiler flags may be needed. This guide focuses on building the export template yourself with a recent version of Emscripten.  
As of 2022-09-04 you also need to use a Rust nightly build.

Confirmed working versions are:
* rustc 1.65.0-nightly (8c6ce6b91 2022-09-02)
* Emscripten 3.1.21-git (3ce1f726b449fd1b90803a6150a881fc8dc711da)


### Installing and configuring Emscripten

We will be using the most recent git version of Emscripten (tip-of-tree):

```bash
# Get the emsdk repo
git clone https://github.com/emscripten-core/emsdk.git

# Enter that directory
cd emsdk

# Download and install the tip-of-tree SDK tools.
./emsdk install tot

# Make the "tot" SDK "active" for the current user. (writes .emscripten file)
./emsdk activate tot

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

Set the newly built export template as a [custom template](https://user-images.githubusercontent.com/2171264/175822720-bcd2f1ff-0a1d-4495-9f9c-892d42e9bdcd.png) in Godot and be sure to set the export type as GDNative. When exporting, uncheck "Export With Debug".

### Building your Rust code

In your project's `config.toml`, add the following:

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
	
cargo +nightly build --target=wasm32-unknown-emscripten --release
```

The result is a `.wasm` file that you add in your `GDNativeLibrary` properties under `entry/HTML5.wasm32`, just like you would add a `.so` or a `.dll` for Linux or Windows.

### Errors you might encounter

Compile time:
* `failed to run custom build command for gdnative-sys v0.10.1`, `fatal error: 'wchar.h' file not found`: Emscripten cache not populated, build Godot export template first
* `undefined symbol: __cxa_is_pointer_type`: You need to build with `-Clink-arg=-sSIDE_MODULE=2`

Runtime:
* `indirect call signature mismatch`: Possibly due to Emscripten version mismatch between Godot and Rust
* `need the dylink section to be first`: Same as above
* `WebAssembly.Module(): Compiling function #1 failed: invalid local index`:  You need to build with `-Cpanic=abort`

### Further reading

In the future, some of this will probably not be needed, for more info see:
* [Tracking issue for godot-rust wasm support](https://github.com/godot-rust/godot-rust/issues/647)
* [Rust](https://github.com/rust-lang/rust/issues/98155) and [Emscripten](https://github.com/rust-lang/rust/pull/98303#issuecomment-1162172132) issues that explain the need for `-Zlink-native-libraries=no`
* [Emscripten PR that should have obsoleted `-Cpanic=abort`](https://github.com/emscripten-core/emscripten/pull/17328)
* [Rust PR that would obsolete `-Clink-arg=-sSIDE_MODULE=2`](https://github.com/rust-lang/rust/pull/98358)
