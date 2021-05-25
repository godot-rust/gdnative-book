# Rust Panic Handler

When using GDNative, Rust panics are ignored by Godot by default. This recipe can be used to catch those panics in the Godot editor at runtime.

This recipe was written and tested with godot-rust 0.9.3 with Rust version 1.52.1

## GDScript Hook

First create a GDScript with the following code named "rust_panic_hook.gd"

```
extends Node

func rust_panic_hook(error_msg: String) -> void:
	assert(false, error_msg)

```

In the Project Settings -> Autoload menu, create an autoload singleton referencing the script (in this case rust_panic_hook.gd).

Pick a unique name that identifies the autoload singleton. You will need to use this name to find the autoload singleton in Rust.

For this example, we are using the autoload name "rust_panic_hook".

At this point we have our GDScript based  panic hook we can use in Rust.

## GDNative Panic Hook Initialization

In the GDNative library code's entry point (lib.rs by default).

```rust
pub fn init_panic_hook() {
    // To enable backtrace, you will need the `backtrace` crate to be included in your cargo.toml, or 
    // a version of rust where backtrace is included in the standard library (e.g. Rust nightly as of the date of publishing)
    // use backtrace::Backtrace;
    // use std::backtrace::Backtrace;
    let old_hook = std::panic::take_hook();
    std::panic::set_hook(Box::new(move |panic_info| {
        let loc_string;
        if let Some(location) = panic_info.location() {
            loc_string = format!("file '{}' at line {}", location.file(), location.line());
        } else {
            loc_string = "unknown location".to_owned()
        }

        let error_message;
        if let Some(s) = panic_info.payload().downcast_ref::<&str>() {
            error_message = format!("[RUST] {}: panic occurred: {:?}", loc_string, s);
        } else if let Some(s) = panic_info.payload().downcast_ref::<String>() {
            error_message = format!("[RUST] {}: panic occurred: {:?}", loc_string, s);
        } else {
            error_message = format!("[RUST] {}: unknown panic occurred", loc_string);
        }
        godot_error!("{}", error_message);
        // Uncomment the following line if backtrace crate is included as a dependency
        // godot_error!("Backtrace:\n{:?}", Backtrace::new());
        (*(old_hook.as_ref()))(panic_info);

        unsafe {
            if let Some(gd_panic_hook) = gdnative::api::utils::autoload::<gdnative::api::Node>("rust_panic_hook") {
                gd_panic_hook.call("rust_panic_hook", &[GodotString::from_str(error_message).to_variant()]);
            }
        }
    }));
}
```

The details the process in the above code is as follows:
1. Get the default panic hook from Rust
2. Create a new panic hook closure to output to the Godot console
3. Get the location string and error message from the `panic_info` closure parameter and print the message to the console
4. Optionally, retreive and print the backtrace
5. Execute the old panic hook so that the normal panic behavior still occurs
6. Call the function defined on your GDScript panic hook script

The final step is to call `init_panic_hook()` at the end of the `init` function that you pass in the `godot_init(init)` macro such as in the following code.

```rust
// GDnative entrypoint
fn init(handle: InitHandle) {
    // -- class registration above
    init_panic_hook();
}
```

Now you can run your game and once it is fully initialized, any panics will pause the game execution and print the panic message in Godot's editor in the Debugger's Error tab.
