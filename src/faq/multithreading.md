# Multithreading FAQ

## How do I use multithreading?

Follow [Godot's thread safety guidelines](https://docs.godotengine.org/en/stable/tutorials/threads/index.html) and change your settings to "Multi-threaded" in "Project Settings > Rendering > Threading".

## Why does my game freeze while using multiple threads?

Make sure you have multi-threading enabled under "Project Settings > Rendering > Threading".

In addition, you will need to ensure that you are not ending up with a deadlock scenario within your Rust code.

## Why is accessing the servers using multiple threads is slow?

Aside from code Deadlocking issues, there are a few potential points to investigate when using Godot's Servers.
### Potential Cause #1 - Command queue is too small

For multi-threaded access the servers use a thread-safe command queue. This allows you to send multiple commands from any thread. These queues have a fixed size that can be set in the threading section of the Memory -> Limits section of the Project settings.

### Potential Cause #2 - Using rayon parallel iterators

The reference created by a server method such as  `VisualServer::godot_singleton()` is not thread-safe. As such, you have to individually get a reference to the singleton and this can cause some severe slowdown that would not occur if the same job were run from a single thread.

Multi-threading with Servers would require that you manually create a threadpool that can hold the reference to the Server. Additional testing is required to ensure that this is an actual optimization.

## Why do I get `DifferentThread` error when calling my code from multiple threads?

For example, what does the following error indicate?

```rust
ERROR: <native>: gdnative-core: method call failed with error: DifferentThread { original: ThreadId(1), current: ThreadId(2) }
   At: src/path/to/class.rs:29
ERROR: <native>: gdnative-core: check module level documentation on gdnative::user_data for more information
   At: src/path/to/class.rs:29
```

If you are calling certain code from Godot and receiving the following error, it is likely that you need to change the user_data that is implemented for your `NativeClass`. If no type is specified, it defaults to `LocalCellData` which can only be used from the thread it was created in.

See the [official docs](https://docs.rs/gdnative/latest/gdnative/nativescript/user_data/index.html) for more information on when you should use each type of data.
