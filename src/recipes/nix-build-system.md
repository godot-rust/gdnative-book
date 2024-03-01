# Recipe: Nix Flakes as development environment

**Disclaimer**: _Currently the following steps are tested and confirmed to work on NixOS only._

[Flakes](https://nixos.wiki/wiki/Flakes) Flakes is a widelly used experimental feature of managing Nix packages to simplify usability and improve reproducibility of Nix installations.

Please read carefully all this page before starting to use gdnative on NixOS.

This tutorial assumes that you are on a [NixOs](https://nixos.wiki/wiki/Main_Page) system or that Nix is [installed](https://nixos.org/download.html#nix-quick-install).
And so on, you also need to make sure that [Flakes](https://nixos.wiki/wiki/Flakes) is enabled.

To begin with, follow the [Hello, world!](../getting-started/hello-world.md) guide in Getting Started. Note that the full source code for the project is available at https://github.com/godot-rust/godot-rust/tree/master/examples/hello-world. Because we aren't using the default build system explained in [setup](../getting-started/setup.md), you should only be worried about the content of the project rather than the dependencies.

## What we need ?

  Since [godot-rust](https://github.com/godot-rust/gdnative) use [gdnative](https://docs.godotengine.org/en/3.5/tutorials/scripting/gdnative/what_is_gdnative.html) to bind Rust and Godot, we need to install the following dependencies:

  - The [rust-overlay](https://github.com/oxalica/rust-overlay) to include some rusty toolchains like `rustc`, `cargo`, `rustup` and `rls`.
  - [openssl](https://www.openssl.org/) that provide us some cryptographic functions for rust.
  - [pkg-config](https://www.freedesktop.org/wiki/Software/pkg-config/) to manage the compilation of the project.
  - [godot3](https://godotengine.org/) aliased as godot.
  - [clang](https://clang.llvm.org/) to compile and bind rust code to gdnative.

  We will use the unstable version of nixpkgs to get the latest version of all packages and the flakes feature.

  And we need to specify the `LIBCLANG_PATH` and `BINDGEN_EXTRA_CLANG_ARGS` to make the gdnative aware of the headers and the library.


## Specifying dependencies

Now to the Nix part of the tutorial. In your project directory create a `flake.nix` file. Later on, this file will be used by Flakes to define the dependencies of your project. Below are the default content of `flake.nix` to run the sample project. We will also explain it in brief about the meaning each line of code.

```nix
{
  # A string to define some description to the flakes 
  description = "GdNative Rust Shell";
  # Will push all updated references to the deps
  inputs = {
    # Get the last available version of all nix packages
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    # Defines rust overlay so we can use it later on our shell
    rust-overlay.url = "github:oxalica/rust-overlay";
    # Some utils for reproducibility
    flake-utils.url = "github:numtide/flake-utils";
  };
  outputs = { self, nixpkgs, rust-overlay, flake-utils, ... }:
    flake-utils.lib.eachDefaultSystem (system:
    #In flake-utils enabling reproducibility for the local system to be referenced
      let
        overlays = [
          (import rust-overlay)
        ];
        pkgs = import nixpkgs {
          inherit system overlays;
        };
      in
      with pkgs;
      {
        devShells.default = mkShell.override { stdenv = pkgs.clangStdenv; } {
          # Point bindgen to where the clang library would be
          LIBCLANG_PATH = "${pkgs.libclang.lib}/lib";
          # Make clang aware of a few headers (stdbool.h, wchar.h)
          BINDGEN_EXTRA_CLANG_ARGS = with pkgs; ''
            -isystem ${llvmPackages.libclang.lib}/lib/clang/${lib.getVersion clang}/include
            -isystem ${llvmPackages.libclang.out}/lib/clang/${lib.getVersion clang}/include
            -isystem ${glibc.dev}/include
          ''
          # Setup all the packages that we will need to develop with gdnative.
          buildInputs = [
            openssl
            pkg-config
            rust-bin.stable.latest.default
            godot3
          ];
          shellHook = ''
            alias godot="godot3"
          '';
        };
      }
    );
}
```



## Activating the Flakes environment

In the same directory where you created your flake.nix, you can use the command [`nix develop`](https://nixos.wiki/wiki/Development_environment_with_nix-shell#nix-develop) to load our Flakes config and setup everything for your Nix-capable system.

