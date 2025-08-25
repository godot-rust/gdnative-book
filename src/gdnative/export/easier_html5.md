# HTML5 Exports without Recompiling the Engine

## Limitations and Expectations

Before you begin this tutorial, I expect you:

- Have a meaningful grasp of Godot
- Have a working understanding Rust
- Have a Godot3.5.1+Rust project that works for **Linux** builds.
- That project doesn't use threads or can be made to not use threads.
- Have access to a computer running Ubuntu 20.04, a Docker runner, a Virtual Machine.

You can might get a way with other Ubuntu versions, but Godot3.5.1 built their export templates with an Ubuntu 20.04 Docker Container, so I'll use 20.04 for this tutorial.

## Setup Environment

You have two options. If your build environment uses Docker, you can just use my `dockerfile` included at the [bottom of this tutorial](#Docker). Build that image and start a container with your project mounted. Then you can skip to [#Building](#building).

If you can't or do not want to use docker, you'll have to configure your build environment manually. Just continue reading from here.

### Install Packages

You probably already have some of these packages, but in case you don't, here is what you'll need to install after a fresh Ubuntu Install

```bash
apt-get update
apt-get install git curl mingw-w64 libclang-dev clang
```

> mingw-w64 probably isn't strictly needed for a purely WASM build, but I included it here because I use it in my dev machine to compile Windows (as well as WASM and Linux). And you might need it to build Windows targets too. If not, it's probably safe to skip.

### Install Rust

You've probably already done this already, but incase you haven't (or your building your own setup script), here it is.

```bash
curl https://sh.rustup.rs -sSf | sh -s -- --default-toolchain stable -y
```

### Install Rust+Emscripten toolchain

This is not Emscripten itself. This is Rust compiler toolchain for Emscripten. We will install Emscripten next.

```bash
rustup update
rustup update nightly
rustup target install wasm32-unknown-emscripten --toolchain nightly
```

### Install Emscripten

Emscripten does not guarantee compatibility between versions. Because of that, you need to figure out the exact version Godot used to compile the HTML5 export template and use that exact version of Emscripten. Luckly, that's actually really easy to figure out since Godot uses GitHub Actions to compile their export templates.

For `Godot 3.5.1`, we're going to use `Emscripten 3.1.14`, but if you're reading this from the future, you'll need to update this. Go to [Godot's GitHub page](https://github.com/godotengine/godot), find the **TAG** (not branch) of the Godot version your using (should be something like `3.5.1-stable`), then find file `.github/workflows/javascript_builds.yml` (unless they move it). Look for `EM_VERSION: ...` and use that version.

```bash
export EMSCRIPTEN_VERSION=3.1.14
git clone http://github.com/emscripten-core/emsdk.git
cd emsdk
./emsdk install ${EMSCRIPTEN_VERSION}
./emsdk activate ${EMSCRIPTEN_VERSION}
```

Then you need to create some environment variables. These variables need to be set EVERY TIME before you build your project. You can make them part of your working environment (add to `.bashrc` or just run them every time you build). You could use something like the below, but with fixing `EMSDK` to match the fully qualified path of where you ran `git clone...` from the above command.

```bash
export EMSDK="" # YOU have to set this!
source "${EMSDK}/emsdk_env.sh"
export C_INCLUDE_PATH=${EMSDK}/upstream/emscripten/cache/sysroot/include
```

#### Re: Other Emscripten Versions

If you want/need to a specific Emscripten version, you'll have to compile your own HTML5 export template.

If you want/need to recompile Godot and export templates (ie, you have custom Modules, etc), you might decide to use a newer version Emscripten as well.

In either case, your export template and your Godot-Rust compile must use the **same** version of Emscripten.

## Building

Assuming you now have a valid environment (or you're using the included dockerfile) you're ready to set up your Rust+Godot project to build for HTML5.


### Rust->WASM

First, you will need to add this file to your project:

```toml
#.cargo/config.toml
[target.wasm32-unknown-emscripten]
rustflags = [
    "-Clink-arg=-sSIDE_MODULE=2", # build a side module that Godot can load
    "-Zlink-native-libraries=no", # workaround for a wasm-ld error during linking
    "-Cpanic=abort", # workaround for a runtime error related to dyncalls
]
```

Then you should be able to run this:

```bash
cargo +nightly build --target=wasm32-unknown-emscripten --release
```

If you get some build errors here, there is likely something wrong with your build environment. Make sure your environment is correct. 


```bash
$ emsdk list | grep -i installed
         3.1.14    INSTALLED
    (*)    node-14.18.2-64bit           INSTALLED
$ echo $C_INCLUDE_PATH
#this output should be a valid path in the emsdk that looks something like `.../upstream/emscripten/cache/sysroot/include`
```

See the above Emscripten section is any of these don't look correct.

### WASM->Godot

Once it's built, you can add the `.wasm` binary to your GDNativeLibrary. Do this the same way you did for your other build targets (Windows, Linux, etc). You'll find the Rust+WASM binary at `${RUST_PROJECT_PATH}/target/wasm32-unknown-emscripten/release/*.wasm`.

Then you need to add a HTML5 export target. `Project->Export->Add->HTML5`. Then in options there is an `Export Type:` drop down that you'll change to `GDNative`.


If you've done everything correctly, you should be able to run your project in a browser! There should now be a HTML5 logo in the top right corner of the Godot editor. Try it!

## Notes

- WASM, HTML5, and Emscripten are often used pretty interchangeably. They aren't exactly the same. Here is a quick explanation.
    - HTML5 is what Godot calls web build targets.
    - WASM or **W**eb **ASM**bley is is the build target for running low level (ish) code in a web browser
    - Emscripten adds a bunch of tools on top of WASM to make porting projects to a browser easier. It's required for compiling Godot and Godot-Rust on a browser.


### Docker

I use this dockerfile. If you know Docker, this is likely the easiest way to get going. Here is a simple example:

```bash
docker build . \
    -t gdnative_wasm_tutorial \
    -f docker/gdnative_rust_emscrpten.dockerfile \
    --build-arg EMSCRIPTEN_VERSION="3.1.14"
docker run -it \
    -v $(pwd):/project \
    -w /project/rust \
    gdnative_wasm_tutorial \
        cargo +nightly build --target=wasm32-unknown-emscripten --release
```

And the docker file:

```dockerfile
FROM ubuntu:20.04

#small container usablity apps
RUN apt-get update && apt-get install -y \
    zip \
    xz-utils \
    wget \
    curl \
    unzip \
    nano \
    p7zip-full \
    && rm -rf /var/lib/apt/lists/*

#apps required to build Godot-Rust on Windows, Linux, and WASM.
RUN apt-get update && apt-get install -y \
    git \
    curl \
    mingw-w64 \
    libclang-dev \
    clang \
    && rm -rf /var/lib/apt/lists/*

#install rust
RUN curl https://sh.rustup.rs -sSf | sh -s -- --default-toolchain stable -y

#set up rustup nightly for emscripten (idk why the PATH is fucked?)
RUN /root/.cargo/bin/rustup update \
    && /root/.cargo/bin/rustup update nightly \
    && /root/.cargo/bin/rustup target install wasm32-unknown-emscripten --toolchain nightly

#Godot 3.5 exports use Emscripten 3.1.10
#Godot 3.5.1 exports use Emscripten 3.1.14
ARG EMSCRIPTEN_VERSION=3.1.14


#install emscripten
RUN git clone http://github.com/emscripten-core/emsdk.git \
    && cd emsdk \
    && ./emsdk install ${EMSCRIPTEN_VERSION} \
    && ./emsdk activate ${EMSCRIPTEN_VERSION} \
    && rm -rf .git \
    && echo 'source "/emsdk/emsdk_env.sh"' >> $HOME/.bashrc

#env path needed for rust-emscripten
ENV C_INCLUDE_PATH=/emsdk/upstream/emscripten/cache/sysroot/include
ENV PATH="/emsdk:/emsdk/upstream/emscripten:/emsdk/node/14.18.2_64bit/bin:/root/.cargo/bin:${PATH}"
```
