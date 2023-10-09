# Recipe: Custom node plugin

While GDNative in Godot 3.x has some limitations, creating an `EditorPlugin` that registers custom nodes can be used to help integrate your types more tightly in the engine. Using a Custom Nodes plugin, you may register as many `NativeClass` scripts as you need in a single plugin.

1. Create the directory "res://addons/my_custom_nodes/".
2. In the directory "res://addons/my_custom_nodes/", create `plugin.cfg` and `add_my_custom_nodes.gd` and include the following code for each.

## plugin.cfg
```ini
[plugin]
name="My Custom Nodes"
description="Adds my custom nodes for my native classes."
author="your-name"
version="0.1.0"
script="add_my_custom_nodes.gd"
```

## add_my_custom_nodes.gd
```gdscript
tool
extends EditorPlugin

func _enter_tree():
    # Get the base icon from the current editor interface/theme
    var gui = get_editor_interface().get_base_control()
    var node_icon = gui.get_icon("Node", "EditorIcons")
    
    # Alternatively, preload to a custom icon at the following
    # var node_icon preload("res://icon_ferris.png")

    add_custom_type(
        "MyNode",
        "Control",
        preload("res://gdnative/MyNode.gdns"),
        node_icon
    )

    # Add any additional custom nodes here here.

func _exit_tree():
    remove_custom_type("MyNode")
    # Add a remove for each registered custom type to clean up
```

From the editor's main menu, find "Project > Project Settings > Plugins". Find "My Custom Nodes" in the list of plugins and activate this. From here, you can create these directly from the "Create New Node" dialog. In addition, any nodes created this way will disallow the script from being changed or removed.

For more information about this kind of plugin, please refer to the [Godot Documentation](https://docs.godotengine.org/en/stable/tutorials/plugins/editor/making_plugins.html#a-custom-node).

**Note**: This method only works for types that inherit from [`Script`](https://docs.godotengine.org/en/stable/classes/class_script.html). In addition, changing the path to the ".gdns" file will cause the custom `Node` to no longer register correctly.

**Note 2**: This does not register your custom classes in the class database. In order to instantiate the script from GDScript, it will still be necessary to use `var ScriptType = preload("res://path/to/my_node.gdns")` before attempting to instantiate it.
