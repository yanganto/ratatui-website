---
title: Configurable Keybindings
sidebar:
  order: 11
  label: Configurable Vim Keybind Config
---
# Recipe: Configurable Keybindings in Ratatui Apps

This recipe presents a modular and extensible solution, based on the new [ratatui-keybind-template](https://github.com/yanganto/ratatui-keybind-template), to add customizable, user-driven keybindings to your Ratatui application. The system uses the [`crossterm-keybind`](https://github.com/yanganto/crossterm-keybind) crate for TOML-based configuration, backward compatibility, and patch-style overrides.

## Problem statement and motivation

With growing userbases, developers of Terminal UI (TUI) apps often get requests for alternative keybinding schemes (like vim-style bindings or personalized shortcuts). Manually supporting such requests quickly becomes a maintenance burden, and as your app evolves, users expect their custom keybinds to remain compatible across updates.


## Design and Constraints

### Core Pattern

The main idea is to define all keybindings in _a single enum_, use attribute macros to declare default shortcuts, and support external TOML configuration for overrides and patches. 

**The Enum Example**:

```rust
use crossterm_keybind::KeyBind;

#[derive(KeyBind)]
pub enum KeyEvent {
    /// Quit the application
    #[keybindings["Control+c", "Q", "q"]]
    Quit,

    /// Show help menu (via 'h' or F1)
    #[keybindings["h", "F1"]]
    ShowHelp,
}
```

#### How to capture a user input

Within a abstraction, the enum, we do not want to directly compairing the `KeyCode`, `KeyModifiers` of a `crossterm::KeyEvent` when capturing an user's input.  Instead, we pass reference of it to `match_any` method, which is written by the KeyBind derive macro.  Before you match any keyevent, you should initialize first with `KeyEvent::init_and_load(...)`, because it is possible for your users to have cusmized keybinds, this will be explained in the next section.  It can be initalized without any user cusmized keybind by `KeyEvent::init_and_load(None)`.  Normally, you can run it as first task of the main function.

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    KeyEvent::init_and_load(None)?;
}
```

```rust
if KeyBindEvent::Quit.match_any(&key) {
  // Close the app 
} else if KeyBindEvent::ShowHelp.match_any(&key){
  // Show documents 
}
```


#### How can user customize their keybinds

The KeyBind macro also provide a key config example file for user, so user can easy to customize their keybinds.
- `KeyBindEvent::toml_example()` will return the content of the example.
- `KeyBindEvent::to_toml_example(path)` will write the example into a file.

In general, you can have some subcommand and help user to generate the config file to their disk.
Note: You can following other recipe to handle config folder and path, current exmaple is store file in current folder.

```rust
KeyBindEvent::to_toml_example("keybind.toml")
```

**The Config Content Example**:
```toml
# The app will be closed with following key bindings
# - combin key Control and c
# - single key Q
# - single key q
quit = ["Control+c", "Q", "q"]

# A toggle to open/close a widget show all the commands
toggle_help_widget = ["F1", "?"]
```

As you can see, the document of enum will also be included into the config files, so you dont need to write the same thing twice.  User can cusomized the keybind as their needed, you just pass the path of the config file when init keybindings.
```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    KeyEvent::init_and_load("keybind.toml")?;
}
```

If the user only cusmized part of key config, we will patch the user's cusmized on the default one.  You can learn it in details with followin using case.
**
**The Content of User's Config**:
```toml
quit = ["Control+q"]
```
The config can be loaded successfully.
After loaded, the only `Control+q` can quit the application, and the default keys, `Control+c`, `Q`, `q` will not work any more.
The keybinds to open a widget will keep the same as default, because user do not customize it, so user still can use `F1` or `?` to open the widget.
You also get the benifit for backward compatibility of key config, if you only make additions on key binding enum.

#### How can user know current key

When the tui application can be cusmized keybindings, it will be nice to hint user what is the current key binding.  So you can use `fn key_bindings_display()`  for this purpose.
```
println!(
    "type {} for help",
    KeyEvent::ShowHelp.key_bindings_display()
);
```

### Summary

With this approach, the following features are supported:
- **User Customization:** Let users adapt the app to their muscle memory and workflows.
- **Maintainability:** Adding new actions or keys shouldn’t break old configs.
- **Upgradeability:** Users can partially override configs, even as your keybindings evolve.
- **Multiple Shortcuts:** Map several key combos to a single action.
- **Backward Compatibility:** it can always compatible with legacy config, if we only make addition on the Enum.
- **Better User Experience:** Power users and international users can adjust keyboard layouts as needed.

There are some constraints with this approach you need to know a head.
- Always use the enum for new Key Bindings, do not directly handle keycode in functions
- Only addition on the enum to keep keybind config backward compatibility.
- Using macro will slightly increase compiling time, but not easy to detect with mordan computers.

This feature are exactly the same as [gitui](https://github.com/gitui-org/gitui), the difference is gitui using RON format and now you are using TOML format.  The toml example feature are exactly the same as [kld](https://github.com/kuutamolabs/kld). The KeyBind derive macro combined these become a total solution.  It still possible to use `crossterm-keybind-core` only to have a similar idea but different pattern.

## Migration guides for an existing applications

You do not need to worried about the application will broken if some keybind are not mirgrated into enum.  Following are the guide to help you complete the migration without issues.

- Create a keybind enum first, and initialize in the head of main 
  - you can use different naming of enum to avoid confusing, for example `AppEvent`, not `KeyEvent`.
  - using `AppEvent::init_and_load(None)?` first
- Gradurally move crossterm::KeyEvent into the `match_any` of the enum
  - normally the condition will from `match` arms to `if` arms in this step
  - simple search `KeyCode`, `KeyModifiers` is good enough rather than search `KeyEvent`
- Make sure `crossterm::KeyCode` or `crosstem::KeyModifiers` are not using in your project
  - If `KeyCode` and `KeyModifiers` are not directly using, and manager by the KeyBind enum
- Allow user to cusomize the keybind
  - save the key config to disk with `AppEvent::to_toml_example("keybind.toml")`
  - using `AppEvent::init_and_load("("keybind.toml")?` first

## Extras a starter template

### Option1. Using GitHub Template

Click the lefttop green `Use this template` button of [ratatui-keybind-template](https://github.com/yanganto/ratatui-keybind-template).

Setup your project name.


### Option 2. Clone from a GitHub Template

__Begin your project from [ratatui-keybind-template](https://github.com/yanganto/ratatui-keybind-template):

```bash
git clone https://github.com/yanganto/ratatui-keybind-template.git
cd ratatui-keybind-template
cargo run
```

## References

- [ratatui-keybind-template](https://github.com/yanganto/ratatui-keybind-template)
- [crossterm-keybind crate](https://github.com/yanganto/crossterm-keybind)
- [Pull request discussion/background](https://github.com/ratatui/templates/pull/124)

With this approach, you can let contributors and users maintain their own keyboard preferences, reducing maintenance burden and increasing adoption of your Ratatui-based apps.
