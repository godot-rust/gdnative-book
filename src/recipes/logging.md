# Logging

Logging in Godot can be accessed by using the `godot_print!`, `godot_warn!`, and `godot_error!` macros.

These macros only work when the library is loaded by Godot. They will panic instead when invoked outside that context, for example, when the crate is being tested with cargo test

## Simple wrapper macros for test configurations

The first option that you have is wrap the `godot_print!` macros in the following macro that will use `godot_print!` when running with Godot, and stdout when run during tests.

```rust
/// prints to Godot's console, except in tests, there it prints to stdout
macro_rules! console_print {
    ($($args:tt)*) => ({
        if cfg!(test) {
            println!($($args)*);
        } else {
            gdnative::godot_print!($($args)*);
        }
    });
}
```

## Using a logging Crate

A more robust solution is to integrate an existing logging library.

This recipe demonstrates using the `log` crate with `flexi-logger`. While most of this guide will work with other backends, the initialization and `LogWriter` implementation may differ.

First add the following crates to your `Cargo.toml` file. You may want to check the crates.io pages of the crates for any updates.

```toml
log = "0.4.14"
flexi_logger = "0.17.1"
```

Then, write some code that glues the logging crates with Godot's logging interface. `flexi-logger`, for example, requires a `LogWriter` implementation:

```rust
use gdnative::prelude::*;
use flexi_logger::writers::LogWriter;
use flexi_logger::{DeferredNow, Record};
use log::{Level, LevelFilter};

pub struct GodotLogWriter {}

impl LogWriter for GodotLogWriter {
    fn write(&self, _now: &mut DeferredNow, record: &Record) -> std::io::Result<()> {
        match record.level() {
            // Optionally push the Warnings to the godot_error! macro to display as an error in the Godot editor.
            flexi_logger::Level::Error => godot_error!("{}:{} -- {}", record.level(), record.target(), record.args()),
            // Optionally push the Warnings to the godot_warn!  macro to display as a warning in the Godot editor.
            flexi_logger::Level::Warn => godot_warn!("{}:{} -- {}",record.level(), record.target(), record.args()),
            _ => godot_print!("{}:{} -- {}", record.level(), record.target(), record.args())
        }        ;
        Ok(())
    }

    fn flush(&self) -> std::io::Result<()> {
        Ok(())
    }

    fn max_log_level(&self) -> LevelFilter {
        LevelFilter::Trace
    }
}
```

For the logger setup, place the code logger configuration code in your `fn init(handle: InitHandle)` as follows.

To add the logging configuration, you need to add the initial configuration and start the logger inside the init function.
```rust
fn init(handle: InitHandle) {
    flexi_logger::Logger::with_str("trace")
        .log_target(flexi_logger::LogTarget::Writer(Box::new(crate::util::GodotLogWriter {})))
        .start()
        .expect("the logger should start");
    /* other initialization work goes here */ 
}
godot_init!(init);
```

### Setting up a log target for tests
When running in a test configuration, if you would like logging functionality, you will need to initialize a log target.

As tests are run in parallel, it will be necessary to use something like the following code to initialize the logger only once. The `#[cfg(test)]` attributes are used to ensure that this code is not accessible outside of test builds.

Place this in your crate root (usually lib.rs)

```rust
#[cfg(test)]
use std::sync::Once;
#[cfg(test)]
static TEST_LOGGER_INIT: Once = Once::new();
#[cfg(test)]
fn test_setup_logger() {
    TEST_LOGGER_INIT.call_once(||{
        flexi_logger::Logger::with_str("debug")
        .log_target(flexi_logger::LogTarget::StdOut)
        .start()
        .expect("the logger should start");
    });
}
```

You can call the above code in your units tests with `crate::test_setup_logger()`. Please note: currently there does not appear to be a tes case that will be called before tests are configured, so the `test_setup_logger` will need to be called in every test where you require log output.

Now that the logging is configured, you can use use it in your code such as in the following sample
```rust
log::trace!("trace message: {}", "message string");
log::debug!("debug message: {}", "message string");
log::info!("info message: {}", "message string");
log::warn!("warning message: {}", "message string");
log::error!("error message: {}", "message string");
```

At this point, we have a logging solution implemented for our Rust based code that will pipe the log messages to Godot.

But what about GDScript? It would be nice to have consistent log messages in both GDScript and GDNative. One way to ensure that is to expose the logging functionality to Godot with a `NativeClass`.

### Exposing to GDScript

> ### Note
> As the rust macros cannot get the GDScript name or resource_path, it is necessary to pass the log target from GDScript.

```rust
#[derive(NativeClass, Copy, Clone, Default)]
#[user_data(Aether<DebugLogger>)]
#[inherit(Node)]
pub struct DebugLogger;

#[methods]
impl DebugLogger {
    fn new(_: &Node) -> Self {
        Self {}
    }
    #[export]
    fn error(&self, _owner: &Node, target: String, message: String) {
        log::error!(target: &target, "{}", message);
    }
    #[export]
    fn warn(&self, _: &Node, target: String, message: String) {
        log::warn!(target: &target, "{}", message);
    }
    #[export]
    fn info(&self, _: &Node, target: String, message: String) {
        log::info!(target: &target, "{}", message);
    }
    #[export]
    fn debug(&self, _: &Node, target: String, message: String) {
        log::debug!(target: &target, "{}", message);
    }
    #[export]
    fn trace(&self, _: &Node, target: String, message: String) {
        log::trace!(target: &target, "{}", message);
    }
}
```

After adding the class above with `handle.add_class::<DebugLogger>()` in the `init` function, you may add it as an Autoload Singleton in your project for easy access. In the example below, the name "game_logger" is chosen for the Autoload singleton:

```gdscript
game_logger.trace("name_of_script.gd", "this is a trace message")
game_logger.debug("name_of_script.gd", "this is a debug message")
game_logger.info("name_of_script.gd", "this is an info message")
game_logger.warn("name_of_script.gd", "this is a warning message")
game_logger.error("name_of_script.gd", "this is an error message")
```

As this is not very ergonomic, it is possible to make a more convenient access point that you can use in your scripts. To make the interface closer to the Rust one, we can create a `Logger` class in GDScript that will call the global methods with the name of our script class.

```gdscript
extends Reference

class_name Logger

var _script_name

func _init(script_name: String) -> void:
	self._script_name = script_name

func trace(msg: String) -> void:
	D.trace(self._script_name, msg)
	
func debug(msg: String) -> void:
	D.debug(self._script_name, msg)

func info(msg: String) -> void:
	D.info(self._script_name, msg)

func warn(msg: String) -> void:
	D.warn(self._script_name, msg)

func error(msg: String) -> void:
	D.error(self._script_name, msg)
```

To use the above class, create an instance of `Logger` in a local variable with the desired `script_name` and use it as in the script example below:

```gdscript
extends Node

var logger = Logger.new("script_name.gd")

func _ready() -> void:
    logger.info("_ready")
```

And now you have a logging solution fully implemented in Rust and usable in GDScript.