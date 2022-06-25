# HTML5

Exporting to HTML5 works just like exporting to other platforms, however there are some things that can make the process a bit tricky and require extra attention.

## General considerations

The Godot HTML5 export templates are built with [Emscripten](https://emscripten.org/), so you need to build your code for the `wasm32-unknown-emscripten` target, `wasm32-unknown-unknown` will not work. Furthermore, you need to use a version of emscripten that is compatible with both your version of Rust and the version of emscripten used to build the export template.  
In practice, this means you might have to [build the HTML5 export template yourself](https://docs.godotengine.org/en/stable/development/compiling/compiling_for_web.html) and/or use a specific version of Rust. You can check which versions you are using like this:

```bash
emcc -v
rustc --version --verbose
```

Also note that `wasm32-unknown-emscripten` is a 32-bit target, which can cause problems with crates incorrectly assuming that `usize` is 64 bits wide.

## Godot 3.4.4

**Disclaimer**: _Currently, the following steps are only tested and confirmed to work on Linux._

Godot 3.4.4's export template is built with a very old Emscripten version that doesn't play nice with Rust. We will have to build it ourselves with a more recent version.  
We also need to build the `std` crate ourselves, so we will have to use a nightly Rust build.

Confirmed working versions are:
* rustc 1.63.0-nightly (dc80ca78b 2022-06-21)
* Emscripten 3.1.15-git (4100960b42619b28f19baf4254d5db2097234b32)


### Getting Emscripten

We will be using Emscripten tip-of-tree:

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

Set the newly built export template as a custom template in Godot and be sure to set the export type as GDNative. When exporting, uncheck "Export With Debug".

### Building your Rust code

In your project's `config.toml`, add the following:

```toml
[target.wasm32-unknown-emscripten]
rustflags = [
	"-Clink-arg=-sSIDE_MODULE=2", # build a side module that Godot can load
	"-Crelocation-model=pic", # needed to prevent linker errors
	"-Cpanic=abort", # panic unwinding is currently broken without -sWASM_BIGINT, see below
	#"-Clink-arg=-sWASM_BIGINT", # alternative to panic=abort, requires building godot with -sWASM_BIGINT also
]
```

Build like this:

```bash
source "/path/to/emsdk-portable/emsdk_env.sh"
export C_INCLUDE_PATH=$EMSDK/upstream/emscripten/cache/sysroot/include
	
cargo +nightly build --target=wasm32-unknown-emscripten --release -Zbuild-std=core,std,alloc,panic_abort
```

### Further reading

In the future, some/all of this will probably not be needed, for more info see:
* [Tracking issue for godot-rust wasm support](https://github.com/godot-rust/godot-rust/issues/647)
* [Rust PR that obsoletes -Clink-arg=-sSIDE_MODULE=2](https://github.com/rust-lang/rust/pull/98358)
* [Rust PR that obsoletes -Crelocation-model=pic and -Zbuild-std](https://github.com/rust-lang/rust/pull/98149)
* [Godot PR that updates Emscripten version](https://github.com/godotengine/godot/pull/61989)
* [Godot PR that would obsolete -Cpanic=abort in favor of -sWASM_BIGINT](https://github.com/godotengine/godot/pull/62397)