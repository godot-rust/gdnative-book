# Godot 4: gdext library

This chapter is under construction and is going to elaborate the **Rust binding for Godot 4**. 



## Currently supported features

For an up-to-date overview of implementation status, consult [issue #24][features].


## GDExtension API: what's new

This section briefly mentions the difference between the native interfaces in Godot 3 and 4 from a functional point of view, 
without going into Rust.

While the underlying FFI (foreign function interface) layer has been completely rewritten, a lot of concepts remain the same
from a user point of view. In particular, Godot's approach with a node-based scene graph, composed of classes in an inheritance
relation, has not changed.

That said, there are some notable differences:

1. **No more native scripts**
   
   With GDNative, Rust classes could be registered as _native scripts_. These scripts are attached to nodes in order to enhance
   their functionality, analogous to how GDScript scripts could be attached. GDExtension on the other hand directly supports Rust types
   as engine classes, see also next point.

   Keep this in mind when porting GDScript code to Rust: instead of replacing the GDScript with a native script, you need to change the
   node type to a Rust class that inherits the node.

2. **First-class citizen types**

   In Godot 3, user-defined native classes had lots of limitations in the editor: type annotations were not fully supported, they could
   not easily be used as custom resources, etc. With GDExtension, user-defined classes in Rust behave much closer to GDScript classes.

3. **Always-on**  
   
   There is no differentiation between "tool" and "normal" scripts anymore; Rust logic runs as soon as the Godot editor launches.
   gdext will likely add an opt-out mechanism to limit execution to the launched game, as that's usually the intended behavior.
   Currently, if you want logic to execute only when the game runs, write this code at the beginning of your process functions:
   ```rust
   if Engine::singleton().is_editor_hint() {
      return;
   }
   ```

4. **No recompilation while editor is open**

   While GDNative allows the Rust library to be recompiled and changes to take effect when the game is launched from the editor, this
   is no longer possible in GDExtension, at least not on all platforms. This limitation is tracked as a Godot bug in [issue #66231].


[features]: https://github.com/godot-rust/gdextension/issues/24
[issue #66231]: https://github.com/godotengine/godot/issues/66231
