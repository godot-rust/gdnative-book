# Compatibility and stability

The gdext library supports all stable Godot releases starting from Godot 4.0. 


## Compatibility with Godot

When developing extension libraries (or just "extensions"), you need to consider which engine version you want to target.
There are two conceptually different versions:

* **API version** is the version of GDExtension against which gdext (and the code of your extension) is compiled.
* **Runtime version** is the version of Godot in which the library built with gdext is run.

The two versions can be different, but there are certain constraints (see [below](#current-guarantees)).

### Philosophy

We take compatibility with the engine seriously, in an attempt to build an ecosystem of extensions that are interoperable with multiple Godot versions.
Nothing is more annoying than updating the engine and recompiling 10 plugins/extensions.

This is sometimes difficult, because:
* Godot may introduce subtle breaking changes of which we are not aware.
* Some changes that are non-breaking in C++ and GDScript are breaking in Rust (e.g. default parameters).
* Using newer features needs to come with a fallback/polyfill for older Godot versions.

We run CI jobs against multiple Godot versions, to get a certain level of confidence that updates do not break compatibility.
Nevertheless, the number of possible combinations is large and only growing, so we may miss certain issues.
If you find incompatibilities or violations of the rules stated below, please let us know.


### Current guarantees

Every extension developed with API version `4.0.x` **MUST** be run with the same runtime version.
* In particular, it is not possible to run an extension compiled with API version `4.0.x` in Godot 4.1 or later.
  This is due to breaking changes in Godot's GDExtension API.

Starting from Godot 4.1 official release, extensions can be loaded by any Godot version, as long as
_runtime version **>=** API version_.
* You can run a `4.1` extension in Godot `4.1.1` or `4.2`.
* You cannot run a `4.2` extension in Godot `4.1.1`.
* This is subject to change depending on how the GDExtension API evolves and how many breaking changes we have to deal with.


### Out of scope

We do **not** invest effort in maintaining compatibility with:

1. Godot in-development versions, except for the latest `master` branch.
   * Not that we may take some time to catch-up with the latest changes, so please don't report issues within a few days after 
     upstream changes have landed.
1. Non-stable releases (alpha, beta, RC).
1. Third-party bindings or GDExtension APIs (C#, C++, Python, ...).
   * These may have their own versioning guarantees and release cycles; and there may be specific bugs to such an integration. 
     If you find an issue with gdext and another binding, reproduce it in GDScript to make sure it's relevant for us.
   * We do however maintain compatibility with Godot, so if integrations go through the engine (e.g. Rust calls a method whose
     implementation is in C#), this should work.
1. Godot with non-standard build flags (e.g. disabled modules).
1. Godot forks or engines running third-party modules. 


## Rust API stability

We are still in a phase where a lot of gdext's foundation needs to be built and refined. As such, **expect breaking changes**.
At the current stage, we believe that making APIs more ergonomic and accessible has priority over long-term stability.
The alternative would be to lock us early into a design corner.

Note that many such breaking changes are externally motivated, for example:
* GDExtension changes in a way that cannot be abstracted from the user.
* There are subtleties in the type system or runtime guarantees that can be modeled in a better, safer way (e.g. typed arrays, RIDs).
* We get feedback from game developers and other users stating that certain workflows are very cumbersome.

Once we get into a more stable feature set, we plan to release versions on crates.io and follow semantic versioning.
