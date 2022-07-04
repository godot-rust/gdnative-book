# HTML5

Exporting to HTML5 works just like exporting to other platforms, however there are some things that can make the process a bit tricky and require extra attention.

## What you need (to know)

The Godot HTML5 export templates are built with [Emscripten](https://emscripten.org/), so you need to build your Rust code for the `wasm32-unknown-emscripten` target (`wasm32-unknown-unknown` will not work).  
You need to build your code with a version of Emscripten that is compatible with your version of Rust and (ideally identical to) the version of Emscripten used to build the export template.  
In practice, this means you might have to:
* install a specific version of Emscripten
* [build the HTML5 export template yourself](https://docs.godotengine.org/en/stable/development/compiling/compiling_for_web.html) 
* use a specific version of Rust

You can check which versions of Emscripten/Rust you are using like this:

```bash
emcc -v
rustc --version --verbose
```

A version mismatch between Rust and Emscripten will probably result in compiler errors, while incompatible Emscripten versions between Rust and Godot will result in cryptic runtime errors such as `indirect call signature mismatch`.

Also note that `wasm32-unknown-emscripten` is a 32-bit target, which can cause problems with crates incorrectly assuming that `usize` is 64 bits wide.

## Godot 3.4.4

**Disclaimer**: _Currently, the following steps are only tested and confirmed to work on Linux._

Godot 3.4.4's export template is built with a very old Emscripten version that doesn't play nice with Rust. You will have to build it yourself with a more recent version.  
As of 2022-07-02 you also need to use a Rust nightly build.

Confirmed working versions are:
* rustc 1.64.0-nightly (f2d93935f 2022-07-02)
* Emscripten 3.1.16-git (e026c0e22a986d92695b2563a1e8ef140b5d8a87)


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
	"-Cpanic=abort", # panic unwinding is currently broken without -sWASM_BIGINT, see below
	#"-Clink-arg=-sWASM_BIGINT", # alternative to panic=abort, requires building godot with -sWASM_BIGINT also
]
```

Build like this:

```bash
source "/path/to/emsdk-portable/emsdk_env.sh"
export C_INCLUDE_PATH=$EMSDK/upstream/emscripten/cache/sysroot/include
	
cargo +nightly build --target=wasm32-unknown-emscripten --release
```

The result is a `.wasm` file that you add in your `GDNativeLibrary` properties under `entry/HTML5.wasm32`, just like you would add a `.so` or a `.dll` for Linux or Windows.

### Further reading

In the future, some/all of this will probably not be needed, for more info see:
* [Tracking issue for godot-rust wasm support](https://github.com/godot-rust/godot-rust/issues/647)
* [Rust PR that would obsolete `-Clink-arg=-sSIDE_MODULE=2`](https://github.com/rust-lang/rust/pull/98358)
* [Godot PR that updates Emscripten version for Godot 3.x](https://github.com/godotengine/godot/pull/61989)
* [Godot PR that would obsolete `-Cpanic=abort` in favor of `-sWASM_BIGINT`](https://github.com/godotengine/godot/pull/62397)
* [Emscripten PR that would obsolete `-Cpanic=abort` without requiring `-sWASM_BIGINT`](https://github.com/emscripten-core/emscripten/pull/17328)