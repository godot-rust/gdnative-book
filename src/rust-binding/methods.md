# Exported methods

In order to receive data from Godot, you can export methods. With the `#[export]` attribute, godot-rust takes care of method registration and serialization. Note that the constructor is not annotated with `#[export]`.

The exported method's first parameter is always `&self` or `&mut self` (operating on the Rust object), and the second parameter is `&T` or `TRef<T>` (operating on the Godot base object, with `T` being the inherited type).

```rust
#[derive(NativeClass)]
#[inherit(Node)]
pub struct GodotApi {
    enemy_count: i32,
}

#[methods]
impl GodotApi {
    fn new(_owner: &Node) -> Self {
        // Print to both shell and Godot editor console
        godot_print!("_init()");
        Self { enemy_count: 0 }
    }
    
    #[export]
    fn create_enemy(
        &mut self,
        _owner: &Node,
        typ: String,
        pos: Vector2
    ) {
        godot_print!("create_enemy(): type '{}' at position {:?}", typ, pos);
        self.enemy_count += 1;
    }

    #[export]
    fn create_enemy2(
        &mut self,
        _owner: &Node,
        typ: GodotString,
        pos: Variant
    ) {
        godot_print!("create_enemy2(): type '{}' at position {:?}", typ, pos);
        self.enemy_count += 1;
    }

    #[export]
    fn count_enemies(&self, _owner: &Node) -> i32 {
        self.enemy_count
    }  
}
```
The two creation methods are semantically equivalent, yet they demonstrate how godot-rust implicitly converts the values to the parameter types (unmarshalling). You could use `Variant` everywhere, however it is more type-safe and expressive to use specific types. The same applies to return types, you could use `Variant` instead of `i32`.
`GodotString` is the Godot engine string type, but it can be converted to standard `String`. To choose between the two, consult [the docs](https://docs.rs/gdnative/latest/gdnative/core_types/struct.GodotString.html).

In GDScript, you can then write this code:
```python
var api = GodotApi.new()

api.create_enemy("Orc", Vector2(10, 20));
api.create_enemy2("Elf", Vector2(50, 70));

print("enemies: ", api.count_enemies())

# don't forget to add it to the scene tree, otherwise memory must be managed manually 
self.add_child(api)
```

The output is:
```
_init()
create_enemy(): type 'Orc' at position (10.0, 20.0)
create_enemy2(): type 'Elf' at position Vector2((50, 70))
enemies: 2
```

## Passing classes

The above examples have dealt with simple types such as strings and integers. What if we want to pass entire classes to Rust?

Let's say we want to pass in an enemy from GDScript, instead of creating one locally. It could be represented by the `Node2D` class and directly configured in the Godot editor. What you then would do is use [the `Ref` wrapper](../gdnative-overview/wrappers.md):
```rust
#[derive(NativeClass)]
#[inherit(Node)]
pub struct GodotApi {
    // Store references to all enemy nodes
    enemies: Vec<Ref<Node2D>>,
}

#[methods]
impl GodotApi {
    // new() etc...

    #[export]
    fn add_enemy(
        &mut self,
        _owner: &Node,
        enemy: Ref<Node2D> // pass in enemy
    ) {
        self.enemies.push(enemy);
    }
  
    // You can even return the enemies directly with Vec.
    // In GDScript, you will get an array of nodes.
    // An alternative would be VariantArray, able to hold different types.
    #[export]
    fn get_enemies(
        &self,
        _owner: &Node
    ) ->  Vec<Ref<Node2D>> {
        self.enemies.clone()
    }
}
```

## Special methods

Godot offers some special methods. Most of them implement [notifications](https://docs.godotengine.org/en/stable/getting_started/workflow/best_practices/godot_notifications.html), i.e. callbacks from the engine to notify the class about a change.

If you need to override a Godot special method, just declare it as a normal exported method, with the same name and signature as in GDScript:
```rust
#[export]
fn _ready(&mut self, _owner: &Node) {...}

#[export]
fn _process(&mut self, _owner: &Node, delta: f32) {...}

#[export]
fn _physics_process(&mut self, _owner: &Node, delta: f32) {...}
```

If you want to change how GDScript's default formatter in functions like `str()` or `print()` works, you can overload the `to_string` GDScript method, which corresponds to the following Rust method:
```rust
#[export]
fn _to_string(&self, _owner: &Reference) -> String {...}
```


## Errors

If you pass arguments from GDScript that are incompatible with the Rust method's signature, the method invocation will fail. In this case, the code inside the method is not executed. An error message is printed on the Godot console, and the value `null` is returned for the GDScript function call.

If code inside your method panics (e.g. by calling `unwrap()` on an empty option/result), the same happens: error message and return value `null`.