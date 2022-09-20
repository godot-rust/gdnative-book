# Recipe: Nix as development environment

**Disclaimer**: _Currently the following steps are tested and confirmed to work on Linux only._

[Nix](https://nixos.org/) is a package manager that employs a pure functional approach to dependency management. Nix packages are built and ran in isolated environments. It makes them more portable, but also harder to author. This tutorial will walk you through the process of setting up a package for Godot, GDNative and Rust with Nix.

This tutorial assumes that Nix is [installed](https://nixos.org/download.html#nix-quick-install) in your system.

To begin with, we are going to create a new project using the [Hello, world!](../getting-started/hello-world.md) guide in Getting Started. Note that the full source code for the project is available at https://github.com/godot-rust/godot-rust/tree/master/examples/hello-world. Because we aren't using the default build system explained in [setup](../getting-started/setup.md), you should only be worried about the content of the project rather than the dependencies.


## Specifying dependencies

Now to the Nix part of the tutorial. In the root directory of the project (where `project.godot` is located), create a new file called `shell.nix`. Later on, this file will be evaluated by Nix to define the dependencies of your project. Below are the default content of `shell.nix` to run the sample project. We will also explain it in brief about the meaning each line of code.

```nix
let
    # Get an up-to-date package for enabling OpenGL support in Nix
    nixgl = import (fetchTarball "https://github.com/guibou/nixGL/archive/master.tar.gz") {};

    # Pin the version of the nix package repository that has Godot 3.2.3 and compatible with godot-rust 0.9.3
    # You might want to update the commit hash into the one that have your desired version of Godot
    # You could search for the commit hash of a particular package by using this website https://lazamar.co.uk/nix-versions
    pkgs = import (fetchTarball "https://github.com/nixos/nixpkgs/archive/5658fadedb748cb0bdbcb569a53bd6065a5704a9.tar.gz") {};
in
    # Configure the dependency of your shell
    # Add support for clang for bindgen in godot-rust
    pkgs.mkShell.override { stdenv = pkgs.clangStdenv; } {
        buildInputs = [
            # Rust related dependencies
            pkgs.rustc
            pkgs.cargo
            pkgs.rustfmt
            pkgs.libclang

            # Godot Engine Editor
            pkgs.godot

            # The support for OpenGL in Nix
            nixgl.nixGLDefault
        ];

        # Point bindgen to where the clang library would be
        LIBCLANG_PATH = "${pkgs.libclang.lib}/lib";
        # Make clang aware of a few headers (stdbool.h, wchar.h)
        BINDGEN_EXTRA_CLANG_ARGS = with pkgs; ''
          -isystem ${llvmPackages.libclang.lib}/lib/clang/${lib.getVersion clang}/include
          -isystem ${llvmPackages.libclang.out}/lib/clang/${lib.getVersion clang}/include
          -isystem ${glibc.dev}/include
        '';

        # For Rust language server and rust-analyzer
        RUST_SRC_PATH = "${pkgs.rust.packages.stable.rustPlatform.rustLibSrc}";

        # Alias the godot engine to use nixGL
        shellHook = ''
            alias godot="nixGL godot -e"
        '';
    }
```

If you get any errors about missing headers, you can use [`nix-locate`](https://github.com/bennofs/nix-index#usage) to search for them, e.g. `nix-locate 'include/wchar.h' | grep -v '^('` (the `grep -v` hides indirect packages), and then add the matching Nix package via the `BINDGEN_EXTRA_CLANG_ARGS` env var like above ([context](https://github.com/NixOS/nixpkgs/issues/52447#issuecomment-853429315)).


## Activating the Nix environment

One of the simplest way to activate the nix environment is to use the `nix-shell` command. This program is installed automatically as you install Nix Package Manager.

First, you need to open the root directory of your project. And then to activate your environment, run `nix-shell -v` into your terminal. The optional `-v` flag in the command will configure the command to be more verbose and display what kinds of things is getting installed. Because this is your first time using `nix-shell` on this particular project, it will take some time to download and install all the required dependencies. Subsequent run will be a lot faster after the installation.

To run the project, first you need to compile the `hello-world` Rust library using `cargo build`. After that, you can open the Godot Engine in your terminal using the command `godot`. As seen in `shell.nix`, this command is actually aliased to `nixGL godot -e` in which Godot will be opened using nixGL instead of opening it directly. After running the default scene, you should be able to see a single `hello, world.` printed in the Godot terminal.
