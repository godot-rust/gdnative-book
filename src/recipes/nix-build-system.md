# Recipe: Nix Flakes as development environment

**Disclaimer**: _Currently the following steps are tested and confirmed to work on NixOS only._

[Flakes](https://nixos.wiki/wiki/Flakes) Flakes is a feature of managing Nix packages to simplify usability and improve reproducibility of Nix installations.

This tutorial assumes that you are on a [NixOs](https://nixos.wiki/wiki/Main_Page) system
or that Nix is [installed](https://nixos.org/download.html#nix-quick-install).
Finally, you also need to make sure that [Flakes](https://nixos.wiki/wiki/Flakes) is enabled.

To begin with, we are going to create a new project using the [Hello, world!](../getting-started/hello-world.md) guide in Getting Started. Note that the full source code for the project is available at https://github.com/godot-rust/godot-rust/tree/master/examples/hello-world. Because we aren't using the default build system explained in [setup](../getting-started/setup.md), you should only be worried about the content of the project rather than the dependencies.


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

If you get any errors about missing headers, you can use [`nix-locate`](https://github.com/bennofs/nix-index#usage) to search for them, e.g. `nix-locate 'include/wchar.h' | grep -v '^('` (the `grep -v` hides indirect packages), and then add the matching Nix package via the `BINDGEN_EXTRA_CLANG_ARGS` env var like above ([context](https://github.com/NixOS/nixpkgs/issues/52447#issuecomment-853429315)).


## Activating the Flakes environment

In the same directory where you created your flake.nix, you can use the command [`nix develop`](https://nixos.wiki/wiki/Development_environment_with_nix-shell#nix-develop) to load our Flakes config and setup everything for your Nix-capable system.
