# Mac OS X

**Disclaimer**: _Currently, the following steps are tested and confirmed to work on Linux only._


## Use case

Exporting for Mac OS X is interesting if:

* you do not have access to Apple hardware
* you want to build from a CI, typically on a Docker image

If you have access to a real Mac, building natively is easier.


## Why is this complex ?

Cross-compiling Rust programs for Mac OS X is as simple as:

```sh
rustup target add x86_64-apple-darwin
cargo build --target x86_64-apple-darwin
```

However to build [gdnative-sys](https://crates.io/crates/gdnative-sys) you
need a Mac OS X C/C++ compiler, the Rust compiler is not enough. More precisely you need
an SDK which usually comes with [Xcode](https://developer.apple.com/xcode/).
For Mac users, this SDK is "just there" but when cross-compiling, it is
typically missing, even if your compiler is able to produce Mac OS X compatible binaries.

The most common error is:

```
fatal error: 'TargetConditionals.h' file not found
```

Installing just this file is not enough, this error is usually a consequence
of the whole SDK missing, so there is no chance you can get a build.

What you need to do is:

* download the SDK
* fix all paths and other details so that it ressembles a Mac OS X environment
* then build with `cargo build --target x86_64-apple-darwin`

Hopefully, the first two steps, downloading the SDK and fixing details,
are handled by a tool called [osxcross](https://github.com/tpoechtrager/osxcross)
which is just about setting up a working C/C++ compiler on Linux.

## Howto

```sh
# make sure you have a proper C/C++ native compiler first, as a suggestion:
sudo apt-get install llvm-dev libclang-dev clang libxml2-dev libz-dev

# change the following path to match your setup
export MACOSX_CROSS_COMPILER=$HOME/macosx-cross-compiler

install -d $MACOSX_CROSS_COMPILER/osxcross
install -d $MACOSX_CROSS_COMPILER/cross-compiler
cd $MACOSX_CROSS_COMPILER
git clone https://github.com/tpoechtrager/osxcross && cd osxcross

# picked this version as they work well with godot-rust, feel free to change
git checkout 7c090bd8cd4ad28cf332f1d02267630d8f333c19
```

At this stage you need to [download and package the SDK](https://github.com/tpoechtrager/osxcross#packaging-the-sdk)
which can not be distributed with osxcross for legal reasons.
[Please ensure you have read and understood the Xcode license terms before continuing](https://www.apple.com/legal/sla/docs/xcode.pdf).

You should now have an SDK file, for example `MacOSX10.10.sdk.tar.xz`.

```sh
# move the file where osxcross expects it to be
mv MacOSX10.10.sdk.tar.xz $MACOSX_CROSS_COMPILER/osxcross/tarballs/
# build and install osxcross
UNATTENDED=yes OSX_VERSION_MIN=10.7 TARGET_DIR=$MACOSX_CROSS_COMPILER/cross-compiler ./build.sh
```

At this stage, you should have, in `$MACOSX_CROSS_COMPILER/cross-compiler`,
a working cross-compiler.

Now you need to tell Rust to use it when linking
Mac OS X programs:

```sh
echo "[target.x86_64-apple-darwin]" >> $HOME/.cargo/config
find $MACOSX_CROSS_COMPILER -name x86_64-apple-darwin14-cc -printf 'linker = "%p"\n' >> $HOME/.cargo/config
echo >> $HOME/.cargo/config
```

After this, your `$HOME/.cargo/config` (not the `cargo.toml` file in your project, this is a different file)
should contain:

```toml
[target.x86_64-apple-darwin]
linker = "/home/my-user-name/macosx-cross-compiler/cross-compiler/bin/x86_64-apple-darwin14-cc"
```

Then, we need to also tell the compiler to use the right compiler and headers.
In our example, with SDK 10.10, the env vars we need to export are:

```sh
C_INCLUDE_PATH=$MACOSX_CROSS_COMPILER/cross-compiler/SDK/MacOSX10.10.sdk/usr/include
CC=$MACOSX_CROSS_COMPILER/cross-compiler/bin/x86_64-apple-darwin14-cc
```

You probably do not want to export those permanently as they are very
specific to building for Mac OS X so they are typically passed at each
call to `cargo`, eg:

```sh
C_INCLUDE_PATH=$MACOSX_CROSS_COMPILER/cross-compiler/SDK/MacOSX10.10.sdk/usr/include CC=$MACOSX_CROSS_COMPILER/cross-compiler/bin/x86_64-apple-darwin14-cc cargo build --release --target x86_64-apple-darwin
```

As a consequence, you do *not* need to put `$MACOSX_CROSS_COMPILER/cross-compiler/bin` in your `$PATH` if
you only plan to export [godot-rust](https://github.com/godot-rust/godot-rust) based programs, as the
binary needs to be explicitly overloaded.


## Exporting

Once your `.dylib` file is built, a standard Godot export should work:

```sh
godot --export "Mac OSX" path/to/my.zip
```

Note that when exporting from a non Mac OS X platform, it is not possible to build a `.dmg`.
Instead, a `.zip` is produced. Again, the tool required to build Mac OS X disk images is
only available on Mac OS X. The `.zip` works fine though, it just contains `my.app`
folder, ready to use.

Double-check your `.dylib` file is there.


## Useful links

* https://github.com/tpoechtrager/osxcross : tool used to install the Mac OS X SDK on Linux
* https://wapl.es/rust/2019/02/17/rust-cross-compile-linux-to-macos.html : a complete tutorial on how to use osxcross
