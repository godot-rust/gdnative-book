# Exporting

Exporting [Godot](https://godotengine.org/) projects using [godot-rust](https://github.com/godot-rust/godot-rust)
is a 2 steps process:

* build your Rust code with cargo, which triggers a build of [gdnative crate](https://crates.io/crates/gdnative) for the target platform;
* do a standard [Godot export](https://docs.godotengine.org/en/stable/getting_started/workflow/export/exporting_projects.html).

If the target you are exporting to is the same as the one you are developping on,
the export is straightforward, however when cross-compiling (eg: exporting for a mobile platform,
or building from a Docker image on a CI)
you need to correctly set up a cross-compiler for the target platform. Rust does this very well,
so provided you only write Rust code, cross-compiling is easy.
However to build [gdnative-sys](https://crates.io/crates/gdnative-sys) you need a working C/C++
cross compiler with, among other things, the correct headers and linker.

How to set up such a cross-compiler depends on the source and the target platform.
