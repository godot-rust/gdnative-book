# Recipe: Loading external resource files

If you need to use any files that aren't explicitly supported by Godot's [Resource Loader](https://docs.godotengine.org/en/stable/classes/class_resourceloader.html), it will be necessary to process and load the file yourself.

This recipe covers two methods of loading these files.

## Option 1 - Embed the File into the Binary

The simplest way is to embed the resource directly into your binary by using the `std::include_bytes` macro.

To embed the file directly into the binary you can use the following macro:

```rust
// To have global immutable access to the file.
const RESOURCE: &'static [u8] = include_bytes!("path/to/resource/file");

fn process_the_resource() {
    // Include the file locally
    let bytes = include_bytes!("path/to/resource/file");
}
```

This can be useful for embedding any information that should be included at build time.

For example: such as if you wish to hard-code certain features like cheat codes, developer consoles, or default user configuration as a file rather than a build flag.

This approach is much more limited as it requires recompiling for all targets whenever changes to the resources are made.

## Option 2 - Embed the File in the PCK

For most other use-cases you can use Godot's PCK file to export your resources into the game.

This can be accomplished by using the `gdnative::api::File` module as follows.

```rust

fn load_resource_as_string(filepath: &str) -> String {
    use gdnative::api::File;
    let file = File::new();
    file.open(filepath, File::READ).expect(&format!("{} must exist", filepath));
    
    let data: GodotString = file.get_as_text();
    // Depending upon your use-case you can also use the following methods depending upon your use-case.
    // let line: StringArray = file.get_csv_line(0);
    // let file_len = file.get_len();
    // let bytes: ByteArray = file.get_bytes(file_len);
    data.to_string()
}
```

See the [`File` Class Documentation](https://docs.rs/gdnative/latest/gdnative/api/struct.File.html) for every function that you use for loading the resources.

After you retrieve the data in the desired format, you can process it like you would normal Rust code.

## Option #3 Save and Load filedata as user_data

Godot allows access to device side user directory for the project under "user://". This works very similar to loading above and it uses the `gdnative::api::File` API

Note: Saving only works on resource paths (paths starting with "res://") when Godot is being run in the editor. After exporting "res://" becomes read-only.

Example on writing and reading string data.

```rust
fn save_data_from_string(filepath: &str, data: &str) {
    use gdnative::api::File;
    let file = File::new();
    file.open(filepath, File::WRITE).expect(&format!("{} must exist", &filepath));
    
    file.store_string(data);
}


fn load_data_as_string(filepath: &str) -> String {
    use gdnative::api::File;
    let file = File::new();
    file.open(filepath, File::READ).expect(&format!("{} must exist", &filepath));
    
    let data: GodotString = file.get_as_text();
    data.to_string()
}
```

For more information on the paths, please refer to the [File System Tutorial](https://docs.godotengine.org/en/3.0/getting_started/step_by_step/filesystem.html#resource-path).


### Testing

This section is for unit testing from Rust without loading Godot. As `gdnative::api::File` requires that Godot be running, any Rust-only unit tests will require a separate method to be implemented in order to load the resources. This can be accomplished by creating separate code paths or functions `#[cfg(test)]` and `#[cfg(not(test))]` attributes to differentiate between test configuration and Godot library configurations.

In test configurations, you will need to ensure that your loading code uses `std::fs::File` or some equivalent to read your load.

### Exporting

When exporting your game, under the `Resources` Tab you will need to add a filter so that godot will pack those resources into the .pck file.

For example: If you are using .json, .csv and .ron files, you will need to use include `*.json, *.csv, *.ron` in the "Filters to Export non-resource files/folders" field.
