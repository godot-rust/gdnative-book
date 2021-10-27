# Tokio Runtime

This recipe is based off of the test written for `gdnative-async`, which uses the `futures` crate in the executor. For cases where you may need a `tokio` runtime, it is possible to execute spawned tokio tasks in much the same way, with some alterations.

## Defining the Executor
The executor itself can be defined the same way.

```rust
thread_local! {
	static EXECUTOR: &'static SharedLocalPool = {
		Box::leak(Box::new(SharedLocalPool::default()))
	};
}
```

However, our `SharedLocalPool` will store a `LocalSet` instead, and the `futures::task::LocalSpawn` implementation for the type will simply spawn a local task from that.

```rust
use tokio::task::LocalSet;

#[derive(Default)]
struct SharedLocalPool {
	local_set: LocalSet,
}

impl futures::task::LocalSpawn for SharedLocalPool {
	fn spawn_local_obj(
		&self,
		future: futures::task::LocalFutureObj<'static, ()>,
	) -> Result<(), futures::task::SpawnError> {
		self.local_set.spawn_local(future);

		Ok(())
	}
}
```

## The Executor Driver

Finally, we need to create a `NativeClass` which will act as the driver for our executor. This will store the tokio `Runtime`.

```rust
use tokio::runtime::{Builder, Runtime};

#[derive(NativeClass)]
#[inherit(Node)]
struct AsyncExecutorDriver {
	runtime: Runtime,
}

impl AsyncExecutorDriver {
	fn new(_owner: &Node) -> Self {
		AsyncExecutorDriver {
			runtime: Builder::new_current_thread()
				.enable_io() 	// optional, depending on your needs
				.enable_time() 	// optional, depending on your needs
				.build()
				.unwrap(),
		}
	}
}
```

In the `_process` call of our `AsyncExecutorDriver`, we can block on `run_until` on the `LocalSet`, which will run or wake any of its local tasks.

```rust
#[methods]
impl AsyncExecutorDriver {
	#[export]
	fn _process(&self, _owner: &Node, _delta: f64) {
		EXECUTOR.with(|e| {
			self.runtime
				.block_on(async {
					e.local_set
						.run_until(async {
							tokio::task::spawn_local(async {}).await
						})
						.await
				})
				.unwrap()
		})
	}
}
```

From there, initializing is just the same as it is in the tests.

```rust
fn init(handle: InitHandle) {
	gdnative::tasks::register_runtime(&handle);
	gdnative::tasks::set_executor(EXECUTOR.with(|e| *e));

	...
	handle.add_class::<AsyncExecutorDriver>();
}
```

