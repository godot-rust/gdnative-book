# Debugging

You can debug your Rust code simply by attaching LLDB to the Godot editor and telling the editor to launch your project. 
However, the process for doing this will vary based on your IDE and platform.

## Launching with VSCode

Here is an example launch configuration for Visual Studio Code (typically found in `./.vscode/launch.json` relative to your project's root). This example assumes you have the [CodeLLDB](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb) 
extension installed, which is common for Rust development.

```jsonc
{
    "configurations": [
        {
            "name": "Debug Project (Godot 4)",
            "type": "lldb",
            "request": "launch",
            "cwd": "${workspaceFolder}",
            "args": [
                "-e", // run editor (remove this if you want to launch the scene directly)
                "-w", // windowed mode
            ],
            "linux": {
                "program": "/usr/local/bin/godot4",
            },
            "windows": {
                "program": "C:\\Program Files\\Godot\\Godot_v4.1.X.exe",
            },
            "osx": {
                // NOTE: on Mac the Godot.app needs to be manually re-signed to enable debugging (see below)
                "program": "/Applications/Godot.app/Contents/MacOS/Godot",
            }
        }
    ]
}
```

## Debugging on MacOS

Attaching a debugger to an executable that wasn't compiled locally (the Godot editor, in this example) requires special considerations on MacOS due to its System 
Integrity Protection (SIP) security feature. 
Even though your extension will be compiled locally, LLDB will be unable to attach to the Godot host process without manual re-signing.

In order to re-sign, simply create a file called `editor.entitlements`[^1] with the following contents (it is recommended to check this file in since each dev will need to re-sign their local install if you have a team):

[^1]: Be sure to use the `editor.entitlements` file below rather than the one from the Godot docs as it includes the required `com.apple.security.get-task-allow` key not currently present in Godot's instructions.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>com.apple.security.cs.allow-dyld-environment-variables</key>
        <true/>
        <key>com.apple.security.cs.allow-jit</key>
        <true/>
        <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
        <true/>
        <key>com.apple.security.cs.disable-executable-page-protection</key>
        <true/>
        <key>com.apple.security.cs.disable-library-validation</key>
        <true/>
        <key>com.apple.security.device.audio-input</key>
        <true/>
        <key>com.apple.security.device.camera</key>
        <true/>
        <key>com.apple.security.get-task-allow</key>
        <true/>
    </dict>
</plist>
```

Then you can run `codesign -s - --deep --force --options=runtime --entitlements ./editor.entitlements /Applications/Godot.app` in Terminal to complete the process.

The re-signing process is covered in more detail in the [Godot Docs](https://docs.godotengine.org/en/stable/contributing/development/debugging/macos_debug.html).
