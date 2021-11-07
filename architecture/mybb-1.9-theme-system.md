# MyBB 1.9 Theme System

## Definitions

*general*

- **Core** — the MyBB forum software in its unmodified state and without Extensions
- **Web Root Directory** — MyBB's main directory (referred to by PHP constant `MYBB_ROOT`)

*extensions*

- **Extension** — a third-party component recognized by the application
  - **Plugin** — a component intended to modify the application's behavior
  - **Theme** — a component defining the user interface
    - **Core Theme** — a Theme belonging to the Core, defining MyBB's default user interface; not editable during normal use
    - **Original Theme** — a Theme maintained by third-party; not editable during normal use
    - **Board Theme** — an editable Theme
  - **Extension Package** — a distributable set of files associated with an Extension
  - **Extension Directory** — a directory containing files from an Extension Package, located directly under `inc/plugins/` or `inc/themes/`
  - **Extension Metadata** — information about the extension, such as the author and license, used in Extension Management
  - **Extension Manifest File** — a `manifest.json` file in JSON format, compatible with [Composer schema](https://getcomposer.org/doc/04-schema.md), containing Extension Metadata, located directly in an Extension Directory
- **Extension System** — the Theme System and the Plugin System
  - **Plugin System** — code responsible for execution of Plugins
  - **Theme System** — code responsible for execution and rendering of Themes

- **Extension Management** — code and events related to configuring Extensions

*interface*

- **Themelet** — Interface Resources with associated properties [Interface Resource Properties]
  - **Interface Resource** — an interface-related file intended to be processed or exposed through HTTP
    - **Template** — code interpreted by a template engine
      - **PHP Template** — a MyBB 1.8-style Template interpreted using `eval()` statements
      - **Twig Template** — a MyBB 1.9-style Template interpreted by Twig
    - **Front-end Asset** — content and files intended to be exposed though HTTP
  - **Interface Resource Property** — data associated with an Interface Resource (e.g. pages to which a stylesheet is attached)
- **Interface Resource Properties File** — a `resources.json` file in JSON format, located directly in a Themelet Directory
- **Themelet Directory** — a directory containing Interface Resources separated into subdirectories named according to their type (e.g. `templates/`, `css/`, `js/`, etc.), and the Interface Resource Properties File

*themes*

- **Theme Property** — data associated with a Theme (e.g. color presets)
- **Theme Properties File** — a `properties.json` file in JSON format containing Theme Properties, located directly in a Theme Definition Directory
- **Theme Definition** — a Themelet associated with a Theme, and Theme Properties
- **Theme Definition Directory** — a directory containing a Theme Definition, associated with a Theme and its version

*theme system*

- **Themelet Inheritance Chain** — an ordered list of Extensions, according to which a final Themelet is resolved, where priority is defined by the number of steps away from the end of the chain
- **Themelet Inheritance Base** — Themelet of the Core Theme and Plugin Themelets of active Plugins
- **Resolved Themelet** — Themelet collected or composed according to a defined Themelet Inheritance Chain
- **Compiled Themelet** — Resolved Themelet prepared for output or final interpretation (e.g. Twig templates converted to PHP code) and web-accessible static files

*macros*

- **Interface Macro** — a set of changes to a Themelet that can be stored, applied, and reversed
  - **Static Interface Macro** — an Interface Macro that operates on a Themelet and causes it to be stored in modified state
  - **Dynamic Interface Macro** — an Interface Macro that operates on a Resolved Themelet and causes a Compiled Themelet to be stored in modified state

*plugin interface*

- **Plugin Interface Definition** — interface-related data supplied by a Plugin
  - **Plugin Themelet** — a Themelet, supplied by a Plugin, that doesn't exist in the Core Theme
  - **Plugin Interface Macro** — an Interface Macro supplied by a Plugin
- **Plugin Interface Macro Directory** — an innermost directory in the `<extension codename>/<extension version>/` structure associated with a specific Extension and its version located in a Plugin Interface Definition Directory, containing a Plugin Interface Macro File
- **Plugin Interface Macro File** — an `extend.php` file defining a Plugin Interface Macro, located in the Plugin Interface Macro Directory
- **Plugin Interface Definition Directory** — an innermost directory in the `interface/<plugin version>` structure located directly in a Plugin's Extension Directory, associated with a Plugin's version, additionally containing a `macros/` subdirectory with Plugin Interface Macro Directories
