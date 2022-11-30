# FAQ: Multithreading

## Table of contents
<!-- toc -->

## How do I use multithreading?

Make sure you read [Godot's thread safety guidelines](https://docs.godotengine.org/en/stable/tutorials/performance/threads/thread_safe_apis.html).

> This is **EXTREMELY IMPORTANT**.

Read the guidelines again. Make sure you fully understand them.    
Anything not explicitly allowed there is a potential minefield.

This cannot be stressed enough, because threading can easily turn a fun gamedev experience into a debugging nightmare. In fact, most occurrences
of undefined behavior (UB) with godot-rust occur not because of FFI, dangling objects or broken C++ code, but because threads are used incorrectly.
By understanding the implications of multithreading, you can save a lot of effort.

A few points are worth highlighting in particular:

1. Do not mix GDScript's `Thread/Mutex` classes with Rust's [`std::thread`](https://doc.rust-lang.org/std/thread) and
   [`std::sync`](https://doc.rust-lang.org/stable/std/sync) modules; they are not compatible.
   If you absolutely must access GDScript threads from Rust, use the correct [`Thread`](https://docs.rs/gdnative/latest/gdnative/api/struct.Thread.html)
   and [`Mutex`](https://docs.rs/gdnative/latest/gdnative/api/struct.Mutex.html) APIs for it.
2. Prefer Rust threads whenever possible. Safe Rust statically guarantees that no race conditions can occur (deadlocks are still possible).
   In practice, this often means that:
   * your game logic (e.g. loading a map) can run in a separate thread, entirely without touching any Godot APIs
   * your main thread is exclusively accessing Godot APIs
   * the two threads can communicate via [channels](https://doc.rust-lang.org/std/sync/mpsc/fn.channel.html).
3. As elaborated in Godot's guidelines, most Godot classes are **simply not thread-safe**. This means you can absolutely not access them
   concurrently without synchronization. Even if things seem to work, you may invoke UB which can manifest in hard-to-find places.
4. It is tempting to think that synchronizing concurrent access to an object through `Mutex` solves threading issues. However, this may not be
   the case: sometimes there are hidden dependencies between _unrelated_ types -- for example, a `Resource` might access a global
   registry, which is currently written by a _different_ object in another thread. Some types internally use shared caches. A lot of these things
   are undocumented, which means you must know Godot's implementation to be sure. And the implementation can always change and introduce a bug in
   the future. 

**TLDR:** Be conservative regarding assumptions about the thread-safety of Godot APIs.    
Rust threads (without Godot APIs) are a very powerful tool -- use them!


## Why does my game freeze while using multiple threads?

Make sure you have multi-threading enabled under _Project Settings > Rendering > Threading_.

In addition, you will need to ensure that you are not ending up with a deadlock scenario within your Rust code.

Last, you may have violated one of the multithreading guidelines and tips in the first section.


## Why is accessing the servers using multiple threads slow?

Aside from deadlock issues, there are a few potential points to investigate when using Godot's servers.

### Potential cause #1 - command queue is too small

For multi-threaded access the servers use a thread-safe command queue. This allows you to send multiple commands from any thread. These queues have a fixed size that can be set in the threading section of the Memory -> Limits section of the Project settings.

### Potential cause #2 - rayon parallel iterators

The reference created by a server method such as  `VisualServer::godot_singleton()` is not thread-safe. As such, you have to individually get a reference to the singleton.
This can cause severe slowdown, that would not occur if the same job were run from a single thread.

Multi-threading with servers requires that you manually create a thread pool that can hold the reference to the server. 
Additional testing is required to ensure that this is an actual optimization.


## Why do I get the `DifferentThread` error?

For example, what does the following error indicate?

```rust
ERROR: <native>: gdnative-core: method call failed with error: 
       DifferentThread { original: ThreadId(1), current: ThreadId(2) }
   At: src/path/to/class.rs:29
ERROR: <native>: gdnative-core: check module level documentation
                 on gdnative::user_data for more information
   At: src/path/to/class.rs:29
```

If you call certain code from Godot and receive the above error, it is likely that you need to change the `user_data` that comes with 
your `NativeClass` derive. If no type is specified, it defaults to `LocalCellData`, which can only be used from the thread it was created in.

See the [official docs](https://docs.rs/gdnative/latest/gdnative/export/user_data) for more information on when you should use each type of data.
