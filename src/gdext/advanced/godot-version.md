# Selecting a Godot version

By default, `gdext` uses the latest stable release of Godot. This is desired in most cases, but it means that you cannot run your extension in
an older Godot version. Furthermore, you cannot benefit from modified Godot versions (e.g. with custom modules).

If these are features you need, this page will walk you through the necessary steps. 
Read [Compatibility and stability] first and make sure you understand the concept of API and runtime versions.


## Older stable releases

Building gdext against an older Godot API allows you to remain forward-compatible with all engine versions >= that version.
(For Godot 4.0.x, [== applies instead of >=][compat-guarantees].)

In a hypothetical example, building against API 4.1 allows you to run your extension in Godot 4.1.1, 4.1.2 or 4.2.

To choose a version (here `4.0`), add the following to your top-level (workspace) `Cargo.toml`:

```toml
[patch."https://github.com/godot-rust/godot4-prebuilt".godot4-prebuilt]
git = "https://github.com//godot-rust/godot4-prebuilt"
branch = "4.0"
```

(If you're interested in the `//` workaround, see <https://github.com/rust-lang/cargo/issues/5478>).


## Custom Godot versions

If you want to freely choose a Godot binary on your local machine from which the GDExtension API is generated, you can use the Cargo feature
`custom-godot`. If enabled, this will look for a Godot binary in two locations, in this order:

1. The environment variable `GODOT4_BIN`.
2. The binary `godot4` in your `PATH`.

Generated code inside the `godot::engine` and `godot::builtin` modules may now look different from stable releases.
Note that we do not give any support or compatibility guarantees for custom-built GDExtension APIs.

Note that this requires the `bindgen`, as such you may need to install the LLVM toolchain.
Consult the [setup page][setup-llvm] for more information.



[Compatibility and stability]: compatibility.md
[compat-guarantees]: compatibility.md#current-guarantees
[setup-llvm]: ../intro/setup.md#llvm

