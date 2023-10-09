# FAQ: Versioning and supported platforms

## Table of contents
<!-- toc -->

## What does godot-rust's version mean?

godot-rust follows [Cargo's semantic versioning](https://doc.rust-lang.org/cargo/reference/semver.html) for the API breaking changes.


## Is godot-rust stable?

godot-rust will not be considered stable until MAJOR version 1.x.x. As such, MINOR version upgrades are subject to potential API breaking changes, but PATCH versions will not break the API.


## Does godot-rust support Godot 4?

See [dedicated FAQ Page](../faq/godot4.md).


## What is the scope of the godot-rust project?

**Similar Questions**

Why is X feature not available in the API?
Why does X function take Y parameters and return Z?

**Answer**

The short answer is "that is how the Godot API works". As godot-rust is focused on the bindings to Godot, we follow the API.

The longer answer is that the goal of the bindings is to ease the friction between using Rust and interacting with the Godot API. This is accomplished by use of Procedural Macros as well as clever use of traits and trait conversions to keep friction and `unsafe` blocks to a minimum.

Types, which are not part of Godot's `Object` class hierarchy, have an implementation in Rust. This avoids FFI overhead and allows code to be more idiomatic. Examples are `Vector2`, `Vector3`, `Color`, `GodotString`, `Dictionary`, etc.


##  Will godot-rust speed up my game in Godot?

Short Answer: It depends on what you are building and the bottlenecks, but it's pretty fast(tm).

Longer Answer: Generally speaking, compared to GDScript, Rust will (in most cases) be a large performance increase.

Compared to an C or C++ GDNative, the performance should be similar.

Compared to a C++ Engine module, the performance will probably be somewhat slower as GDNative modules cannot take advantage of certain optimizations and need to use the C Foreign Function Interface at runtime.

Caveat: The above apply only to release builds with appropriate release settings. When comparing to debug builds in Rust can be very slow.

Caveat 2: It is still possible to write Rust code that has poor performance. You will still need to be mindful of the specific algorithms that you are choosing.

Caveat 3: A lot of code in a game is not performance-critical and as such, Rust will help only in limited ways. When deciding between GDScript and Rust, you should also consider other factors such as static type safety, scalability, ecosystem and prototyping speed. Try to use the best of both worlds, it's very well possible to have part of your code in GDScript and part in Rust.


## Which platforms are supported?

Short Answer: Wherever you can get your code (and Godot) to run. :)

Long Answer: Rust and Godot are natively available for Windows, Linux, Android and iOS export targets.

As of writing, WASM targets are a tentatively doable, but it requires additional configuration and is currently not supported out of the box. The godot-rust bindings do not officially support this target. The progress in this regard can be tracked in [godot-rust/godot-rust#647](https://github.com/godot-rust/godot-rust/issues/647).


## Does godot-rust support consoles?

The official position of the godot-rust project is that, we do not have the ability to offer console support due to a lacking access to the SDK.

As for whether or not it is possible to compile Rust to run on a console, due to the Non-disclosure Agreements that are a part of the console SDKs, it is unlikely that anyone can give you a definitive answer.

The official Godot documentation [goes into more details](https://docs.godotengine.org/en/stable/tutorials/platform/consoles.html) with how this works from a Godot perspective. Please keep in mind that this does not necessarily extend to GDNative.

Regarding whether Rust can run on a console or not, as most modern consoles are using x86 processor architectures, it should be possible to compile for them. The primary issue will be whether the console manufacturer has a version of the Rust standard library that is building for their console. If not, it would be necessary to port it or leverage a port from another licensed developer.