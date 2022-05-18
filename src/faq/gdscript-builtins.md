# GDScript method alternatives

Godot's GDScript has several [built-in methods](https://docs.godotengine.org/en/stable/classes/class_@gdscript.html) that provide a range of functionality useful for game design and implementation. For Godot 3.x's [GDNative interface](https://github.com/godotengine/godot-headers), these methods are not exposed, but fret not! Most of the math related functions (and many other useful ones) are part of Rust's `std` crate. 

Many of the other commonly needed built in methods have analogs provided by godot-rust, such as [`godot_print!`](https://godot-rust.github.io/docs/gdnative/log/macro.godot_print.html) and [`GodotObject::from_instance_id()`](https://godot-rust.github.io/docs/gdnative/object/trait.GodotObject.html#method.from_instance_id).

This page hopes to provide a quick reference for finding equivalent functionality for these various methods, which can be particularly useful for porting GDScript code to Rust.

## Table of contents
<!-- toc -->

## Maths

Since GDScript's [`float` built-in type](https://docs.godotengine.org/en/stable/classes/class_float.html#class-float) is a double-precision floating point, the references below refer to Rust's the 64 bit versions in `std` ([f64 primitive](https://doc.rust-lang.org/stable/std/primitive.f64.html) and [std::f64](https://doc.rust-lang.org/stable/std/f64/index.html)), 32-bit versions of all of these are also provided by [f32](https://doc.rust-lang.org/stable/std/primitive.f32.html) / [std::f32](https://doc.rust-lang.org/stable/std/f32/index.html).

Though the `float` type in GDScript is a double-precision floating-point, single-precision floating-points are also used as mentioned in the [Godot Documentation](https://docs.godotengine.org/en/stable/classes/class_float.html#class-float):

> Most methods and properties in the engine use 32-bit single-precision floating-point numbers instead, equivalent to `float` in C++, which have 6 reliable decimal digits of precision. For data structures such as [`Vector2`](https://docs.godotengine.org/en/stable/classes/class_vector2.html#class-vector2) and [`Vector3`](https://docs.godotengine.org/en/stable/classes/class_vector3.html#class-vector3), Godot uses 32-bit floating-point numbers.

Community members have also shared some other Rust crates and resources that may provide improved mathing in your project:

* [`glam` crate](https://crates.io/crates/glam) - A simple and fast 3D math library for games and graphics.
* [`ultraviolet` crate](https://crates.io/crates/ultraviolet) - This is a crate to computer-graphics and games-related linear and geometric algebra, but _fast_, both in terms of productivity and in terms of runtime performance.
* [Are we game yet? - Math crates](https://arewegameyet.rs/ecosystem/math/) - Linear algebra libraries, quaternions, color conversion and more

### GDScript built-in math methods

* abs ( float s ) -> float
  - [`f64::abs()`](https://doc.rust-lang.org/std/primitive.f64.html#method.abs)
* acos ( float s ) -> float
  - [`f64::acos()`](https://doc.rust-lang.org/std/primitive.f64.html#method.acos)
* asin ( float s ) -> float
  - [`f64::asin()`](https://doc.rust-lang.org/std/primitive.f64.html#method.asin)
* atan ( float s ) -> float
  - [`f64::atan()`](https://doc.rust-lang.org/std/primitive.f64.html#method.atan)
* atan2 ( float y, float x ) -> float
  - [`f64::atan2()`](https://doc.rust-lang.org/std/primitive.f64.html#method.atan2)
* cartesian2polar ( float x, float y ) -> Vector2
* ceil ( float s ) -> float
  - [`f64::ceil()`](https://doc.rust-lang.org/std/primitive.f64.html#method.ceil)
* clamp ( float value, float min, float max ) -> float
  - [`f64::clamp()`](https://doc.rust-lang.org/std/primitive.f64.html#method.clamp)
* cos ( float s ) -> float
  - [`f64::cos()`](https://doc.rust-lang.org/std/primitive.f64.html#method.cos)
* cosh ( float s ) -> float
  - [`f64::cosh()`](https://doc.rust-lang.org/std/primitive.f64.html#method.cosh)
* db2linear ( float db ) -> float
* decimals ( float step ) -> int
* dectime ( float value, float amount, float step ) -> float
* deg2rad ( float deg ) -> float
  - [`f64::to_degrees()`](https://doc.rust-lang.org/std/primitive.f64.html#method.to_degrees)
* ease ( float s, float curve ) -> float
* exp ( float s ) -> float
  - [`f64::exp()`](https://doc.rust-lang.org/std/primitive.f64.html#method.exp)
* floor ( float s ) -> float
  - [`f64::floor()`](https://doc.rust-lang.org/std/primitive.f64.html#method.floor)
* fmod ( float a, float b ) -> float
* fposmod ( float a, float b ) -> float
* inverse_lerp ( float from, float to, float weight ) -> float
* is_equal_approx ( float a, float b ) -> bool
* is_inf ( float s ) -> bool
  - [`f64::is_infinite()`](https://doc.rust-lang.org/std/primitive.f64.html#method.is_infinite)
* is_nan ( float s ) -> bool
  - [`f64::is_nan()`](https://doc.rust-lang.org/std/primitive.f64.html#method.is_nan)
* is_zero_approx ( float s ) -> bool
* lerp ( Variant from, Variant to, float weight ) -> Variant
* lerp_angle ( float from, float to, float weight ) -> float
* linear2db ( float nrg ) -> float
* log ( float s ) -> float
  - [`f64::ln()`](https://doc.rust-lang.org/std/primitive.f64.html#method.ln)
* max ( float a, float b ) -> float
  - [`f64::max()`](https://doc.rust-lang.org/std/primitive.f64.html#method.max)
* min ( float a, float b ) -> float
  - [`f64::min()`](https://doc.rust-lang.org/std/primitive.f64.html#method.min)
* move_toward ( float from, float to, float delta ) -> float
* nearest_po2 ( int value ) -> int
* polar2cartesian ( float r, float th ) -> Vector2
* posmod ( int a, int b ) -> int
* pow ( float base, float exp ) -> float
* rad2deg ( float rad ) -> float
  - [`f64::to_degrees()`](https://doc.rust-lang.org/std/primitive.f64.html#method.to_degrees)
* range_lerp ( float value, float istart, float istop, float ostart, float ostop ) -> float
* round ( float s ) -> float
  - [`f64::round()`](https://doc.rust-lang.org/std/primitive.f64.html#method.round)
* sign ( float s ) -> float
* sin ( float s ) -> float
  - [`f64::sin()`](https://doc.rust-lang.org/std/primitive.f64.html#method.sin)
* sinh ( float s ) -> float
  - [`f64::sinh()](https://doc.rust-lang.org/std/primitive.f64.html#method.sinh)
* smoothstep ( float from, float to, float s ) -> float
* sqrt ( float s ) -> float
  - [`f64::sqrt()`](https://doc.rust-lang.org/std/primitive.f64.html#method.sqrt)
* step_decimals ( float step ) -> int
* stepify ( float s, float step ) -> float
* tan ( float s ) -> float
  - [`f64::tan()`](https://doc.rust-lang.org/std/primitive.f64.html#method.tan)
* tanh ( float s ) -> float
  - [`f64::tanh()`](https://doc.rust-lang.org/std/primitive.f64.html#method.tanh)
* wrapf ( float value, float min, float max ) -> float
* wrapi ( int value, int min, int max ) -> int

### GDScript built-in math constants

* PI
  - [`std::f64::consts::PI`](https://doc.rust-lang.org/stable/std/f64/consts/constant.PI.html)
* TAU
  - [`std::f64::consts::TAU`](https://doc.rust-lang.org/stable/std/f64/consts/constant.TAU.html)
* INF
  - [`f64::INFINITY`](https://doc.rust-lang.org/stable/std/primitive.f64.html#associatedconstant.INFINITY)
* NAN
  - [`f64::NAN`](https://doc.rust-lang.org/stable/std/primitive.f64.html#associatedconstant.NAN)

## Data/Type Conversions

* Color8 ( int r8, int g8, int b8, int a8=255 ) -> Color
* ColorN ( String name, float alpha=1.0 ) -> Color
* bytes2var ( PoolByteArray bytes, bool allow_objects=false ) -> Variant
* char ( int code ) -> String
* convert ( Variant what, int type ) -> Variant
* dict2inst ( Dictionary dict ) -> Object
* inst2dict ( Object inst ) -> Dictionary
* ord ( String char ) -> int
* parse_json ( String json ) -> Variant
* str ( ... ) vararg -> String
* str2var ( String string ) -> Variant
* to_json ( Variant var ) -> String
* var2bytes ( Variant var, bool full_objects=false ) -> PoolByteArray
* var2str ( Variant var ) -> String

## Random Number Generation

* rand_range ( float from, float to ) -> float
* rand_seed ( int seed ) -> Array
* randf ( ) -> float
* randi ( ) -> float
* randomize ( ) -> void
* seed ( int seed ) -> void

## Debugging / Utility / Other

* assert ( bool condition, String message="" ) -> void
* funcref ( Object instance, String funcname ) -> FuncRef
* get_stack ( ) -> Array
* hash ( Variant var ) -> int
* instance_from_id ( int instance_id ) -> Object
* is_instance_valid ( Object instance ) -> bool
* len ( Variant var ) -> int
* load ( String path ) -> Resource
* preload ( String path ) -> Resource
  - See: [FAQ: What is the Rust equivalent of `preload`?](/faq/code.html#what-is-the-rust-equivalent-of-preload)
* print ( ... ) vararg -> void
  - [`gdnative::log::godot_print!()`](https://godot-rust.github.io/docs/gdnative/log/macro.godot_print.html)
* print_debug ( ... ) vararg -> void
  - [`gdnative::log::godot_dbg!()`](https://godot-rust.github.io/docs/gdnative/log/macro.godot_dbg.html)
* print_stack ( ) -> void
* printerr ( ... ) vararg -> void
* printraw ( ... ) vararg -> void
* prints ( ... ) vararg -> void
* printt ( ... ) vararg -> void
* push_error ( String message ) -> void
* push_warning ( String message ) -> void
* range ( ... ) vararg -> Array
* type_exists ( String type ) -> bool
* typeof ( Variant what ) -> int
* validate_json ( String json ) -> String
* weakref ( Object obj ) -> WeakRef
* yield ( Object object=null, String signal="" ) -> GDScriptFunctionState
