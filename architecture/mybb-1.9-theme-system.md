# MyBB 1.9 Theme System

## Introduction

This document presents for discussion the features and design of MyBB 1.9's theme system, including which features are to be prioritised for the very first release of MyBB 1.9.

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

## Features

Following is a list of features for the end system. The following symbol indicates that a feature is to be implemented for the initial release of 1.9: {~}. The other features are to be deferred until a later 1.9 release.

1. Resource management for plugins:

   1.1. {~} Permanent additions:

      This "extension layer" of new resources and their properties - supplied by plugins - conceptually extends the core theme and is inherited by board themes.

   1.2. {~} Permanent modifications:

      Resource modification functions directly modify the existing resources of board themes, similar to how MyBB 1.8's find_replace_templatesets() function works.

   1.3. Dynamic modifications:

      "Template hooks" provide for dynamic, compile-time modifications of resources, potentially using functions corresponding to those of 1.2 above (and see 3 below).

2. Resource tracking:

   Resources added/modified by 1.1 and 1.2 above are tracked, so that additions/modifications can be undone even when the plugins that made them are removed.

3. Fine-grained modifications (see also "Macros" further below):

   Modification functions for 1.2 above provide for detailed stipulation of changes, to support the resource tracking and undo functionality of 2 above.

   For example (lightly edited extract of code provided by Devilshakerz):

   ```php
      // modify existing template
      \MyBB\ExtendTheme\Template('index.twig')
          ->insertBefore(
              '<a href="custom_page.php">Custom Page</a>',
              '<!-- end: navigation -->',
          );
   ```

4. Conditional applicability:

   Any addition or modification of a resource per 1.1 to 1.3 above can be stipulated as applying to certain themes only, at least via the UI, and potentially programmatically too.

5. {~} Retention of prior theme versions:

   Whenever the core theme or an original theme is updated, its previous version is stored, so as to support enhanced updating and conflict resolution (see next).

6. {~} Enhanced updating and conflict resolution:

   During both the upgrade of MyBB core and the upload of a new version of an original theme, MyBB attempts - conceptually via generating and then applying a patch for each change - to automatically update all resources for all board themes, i.e., handling (passing through) prior modifications that had previously been made to board themes either by an extension or by an admin via the ACP. If automatic application of a conceptual patch fails, then the UI guides the admin through the resolution of the conflict.

   All text-based resources, especially including stylesheets, and, ultimately, Javascript files, are to be fully supported via this feature.

7. {~} The encouragement of good inheritance practices:

   The resources of core and original themes are considered immutable (read-only); only board theme resources may be modified. This is enforced via the UI, but can only be encouraged for direct filesystem edits of templates.

8. {~} Continued, but deprecated, eval()-based template support:

   To make plugin migration easier, but only temporarily: to be removed in a later release.

   [To decide:] Opinion on whether this feature should be included is divided; more discussion is needed, and perhaps even a vote.

9. Compatibility flagging:

   The UI supports the marking of a board theme as compatible with the current version of a plugin. This suppresses the UI from otherwise displaying a need for conflict resolution.

10. Resource management for Javascript (JS) files:

    Similar to the existing stylesheet management, with these features:

    10.1. Dependency support: one JS resource can be flagged as depending on another, in which case the latter is loaded automatically if necessary.

    10.2. Automatic combining, minifying, and caching of multiple JS resources, with cache busting.

11. The merge of the FAStyle plugin into core.

12. The packaging of themes as Composer packages:

    An optional extra method of installation for theme developers to utilise, in addition to the standard installation procedure (uploading a file to the ACP).

13. {~} Pre-processed CSS support:

    In addition to stylesheet resources, pre-processed CSS files are also supported.

14. Colour scheme support:

    Made possible via 13 above; includes a colour scheme editor.

15. Macros:

    Consist in a set of modification instructions (also suitable for feature number 2, resource tracking, above), in which a set of changes is associated with a plugin and its version. These would not be limited to resolving conflicts (i.e., acting as a preconfigured resolution), but could also be used for arbitrary customisations.

16. Support for replacing an immutable parent original theme with a mutable board theme:

    An "advanced" option dedicated to theme developers. Conceptually, it might simply involve copying the theme directory and changing theme type.

17. Semantic versioning:

    The best candidate for the version of a parent theme from which to inherit could be stipulated according to semantic versioning, such that, e.g., changes for v1.2 would be used instead of v2 when theme v1.2.3 is installed, and administrators could be presented with a choice based on previewing changes and conflicts relative to both (less likely candidates might have less conflicts, but that might also be coincidental).

## Design notes

The following notes describe the design of the end system. Where a feature above is not referenced below, its design is assumed to be clear enough for a developer to implement without the guidance of this document, or is yet to be determined.

1. Themes, including templates and other resources, are generally to be stored in the filesystem rather than the database, although some minimal information about themes is to be stored in the database (see below).

2. The filesystem structure for plugins is to be exemplified by the following:

```
    inc/
        plugins/
            plugin-name/
                theme/
                    current/
                        templates/
                        stylesheets/
                        theme-changes/
                            core/
                                1.9.1/
                                    extend.php
                                1.9.0/
                                    extend.php
                        manifest.json
                        properties.json
                        resources.json
                    v1/
                        [as for current]
                    [etc]
```

3. The filesystem structure for themes is to be exemplified by the following (the core theme is to be stored with a `theme-name` of `core`):

```
    inc/
        themes/
            theme-name/
                current/
                    templates/
                    stylesheets/
                    manifest.json
                    properties.json
                    resources.json
                v1/
                    [as for current]
                [etc]
```

4. In the filesystem structures exemplified above, the `templates/` and `stylesheets/` directories may contain subdirectories. These subdirectories may be nested. In the case of subdirectories of `templates/`, each one represents a template group to be shown in a expandable/collapsible hierarchy in the ACP interface. [To decide:] Some means of converting the template group as a directory name into a description/heading suitable for the ACP is required; potentially, these conversions could be provided in the `properties.json` file, which for each could stipulate either a hard-coded language string, or a translatable key into `$lang`.

5. Directory names in the above filesystem structure are to be restricted to lowercase alphabetical characters (since on Windows, filenames are case-insensitive, whereas on Linux/UNIX they are case-sensitive), along with digits, dashes, and underscores.

6. The `manifest.json` file must conform to [the composer.json spec](https://getcomposer.org/doc/04-schema.md#json-schema).

7. [Redundant]

8. Each of the following properties should be stipulated in one or another of the JSON files [To decide: which properties should be stored in which files?]:
    - `Name`|`Name translation key` : Either: a human-readable name for the theme or a key into `$lang` to translate the theme's human-readable name.
    - `Version`                     : A string compatible with PHP's version_compare() function.
    - `Author`                      : The theme's author.
    - `Author website`              : The theme author's website.
    - `Type`                        : One of core, original, and board.
    - `Parent theme`                : If not stipulated, then the MyBB core theme is assumed to be the parent.
    - `Parent theme version`        : The last version of the parent theme from which this theme inherits. If not stipulated, then the latest version of the parent theme is assumed.
    - `Stylesheet-page associations`: A list of pages to which each of the theme's stylesheet(s) is attached.

9. In the database table corresponding to 1.8's `mybb_themes`, the following columns will be retained/removed/added (double parentheses indicate a potential but not finalised choice):
    - ( Retained  ) `tid`          : The ID of the activated theme.
    - ( Retained  ) `name`         : The theme's display name. [To decide:] Potentially keep to allow customising. Otherwise derive from the theme's `manifest.json`.
    - ( Added     ) `codename`     : The theme's codename, corresponding to its directory name in the filesystem. Note that the MyBB extension ecosystem, and especially its Extend site, must ensure that this value is unique.
    - ((Discarded)) `pid`          : The ID of the theme's parent. May be discarded given that the parent theme can already be determined via one of the JSON files in the filesystem. [To decide:] Whether or not to actually discard this column, and, if so, which JSON file to move it into.
    - ((Discarded)) `def`          : Whether or not this is the default theme. [To decide:] Whether or not to actually discard this column (and store which theme is default elsewhere, and if so, where). One reason to discard it is because of redundancy and the hypothetical possibility of data conflicts as-is: multiple themes having a `def` value of one.
    - ( Discarded ) `properties`   : Discarded (or deprecated) due to this data now being stored in the `properties.json` file.
    - ( Discarded ) `stylesheets`  : Discarded (or deprecated) due to this data now being stored in the `resources.json` file.
    - ( Retained  ) `allowedgroups`: Instance-specific references to user groups (may later be normalized as a pivot table).

    [To decide:]
    [Editors of this document should feel free to add their own answers alongside Laird's, or to delete his/all answers when a decision is reached]
    - Should the database table include the theme's `version`?

      [Laird's answer: Probably not. We should probably just assume the equivalent of `current/`. My subsequent answer though assumes that `version` _is_ included.]

    - If so, should the system expose an option to change this version?

      [Laird's answer: Probably, but it should probably be accompanied by a strong "Only if you know what you are doing" warning.]

10. Support for installing themes and plugins via Composer is to be deferred (considered, designed, and implemented) for after the initial release of 1.9.

11. CSS pre-processor support. [To decide:] Whether to include this in the initial 1.9 release [@Devilshakerz seems to think: yes], and if so - or ultimately - how it should work.

12. Installation and upgrade of themes and plugins. [To decide:] How this should work. Four main approaches have been proposed, all of which allow for a placeholder directory, provisionally titled `current/`, to represent the latest version of the (plugin) theme's contents:

    12.1. *Version-stamp during distribution*: On distribution, the developer changes the name of the `inc/plugins/plugin-name/theme/current/` directory to `inc/plugins/plugin-name/theme/<version>/` in the distributed package. The forum admin installs the unzipped package directly into the filesystem. MyBB infers the `current/` directory as the `<version>/` directory with the highest version, or otherwise uses the `current/` directory by default if that directoy (a) exists and (2) *is* the directory with the highest version (as determined from its `manifest.json` file).

       - Drawback: Requires build tools or manual intervention (to rename the `current/` directory) prior to distribution.

    12.2. *Version-stamp on detection*: The distributed package contains a `current/` directory, which is installed, after unzipping, directly into the filesystem. When the ACP is visited, MyBB discovers the uploaded `current/` directory, and renames it to `<version>/` where <version> is stipulated in `manifest.json`.

       - Drawback: This seems to preclude developing with a `current/` directory, since that directory would be constantly renamed - at least whenever the ACP was visited.

    12.3. *Version-stamp on archiving*: An extension (plugin/theme) is either (a) (preferably) uploaded via the ACP as a ZIP archive, or (b) (fallback) uploaded unzipped to a `staging/` directory. MyBB then, either upon ZIP upload or auto-detection of the new subdirectory in the `staging/` directory, renames any existing `current/` directory for the extension in question based on its version, and then moves the new `current/` directory into place from the unzipped contents of the new version of the extension.

       - Drawback: Requires a developer, after releasing a version of the theme, and moving on to development of a new version, to manually archive the `current/` directory in his/her working development directory according to its version number per the release.

    12.4. *Archive and retain*: This is a variation on the "Version-stamp on detection" approach, in which the new `current/` directory is archived to its version directory (as pulled out of its manifest file) immediately upon installation (during detection), and is *also* retained *as* the `current/` directory. This means it is fine if admins overwrite that directory (`current/`) upon upgrade, because it is already archived.

       - Drawback: Redundancy in the filesystem with potential for desynchronisation: the `current/` directory and its equivalent versioned directory are - should be - identical, but may desync during development by the plugin's developer in the `current/` directory.

13. Inheritance: Each theme except for `core` inherits from some other theme, defaulting to `core` if not stipulated. Copy-on-write (COW) semantics apply: if a resource provided by a parent theme is not provided in the child theme, then it is taken as the parent's resource; only if/when the resource is overridden by the child theme is the copy written into the child theme.

    13.1. Inheritance and versioning: [To decide:] How this should work. [Whilst @Devilshakerz seems to have some clear ideas on how this should work, the original author of this point (Laird) is not himself all that clear on them, and invites @Devilshakerz to fill in this point with his ideas.]

    13.2. Inheritance via the "extension layer" of feature number 1.1: [To decide:] How is this to be implemented? For example, does some form of COW apply, as for inheriting from parent themes, or are copies of the plugin's interface resources made into each board theme's directories from the start? How are updates to the extension layer handled?

    [To decide:]
    [Editors of this document should feel free to add their own answers alongside Laird's, or to delete his/all answers when a decision is reached]
    Assuming that the `mybb_themes` database table includes the theme's `version`:
    - Should the parent theme information include a version number that, unless bumped, will cause the child theme to continue to inherit from the old version (but issue "attention needed" warnings)?
      [Laird's answer: Probably, yes.]
    - Should changes in parent themes, and in plugins' modification instructions, be propagated silently or manually (following "attention needed" warnings, with possible edit suggestions)?
      [Laird's answer: Upon upload of a new version of a theme (including the core them during an upgrade) or resource-generating plugin, the admin should probably be given, during the upgrade/install procedure, the choice of automatic (except where manual conflict resolution is required) or manual update of board themes per modification instructions. Presumably, admins would generally choose the automatic option.]
    - Should board themes be versioned?
      [Laird's answer: Potentially, but they should at least be correlated with the version of the theme from which they inherit.]
    - If unversioned, should changes to board themes be logged (e.g. in diff format), especially given the automation involved?
      [Laird's answer: Probably, yes, but not in the initial 1.9 release.]

14. Enhanced updating and conflict resolution: The Text_Diff third-party library already in use for MyBB 1.8 could be leveraged, in particular via its support for three-way diffs. It seems that it would need some modifications, but, generally, it could support the following implementation:

    Two panes are presented side by side. The left pane shows the differences for each line between the original version of the resource and the newly released version of the resource (using the same formatting and styling as the current diffs for 1.8). The right pane shows the differences for each line between the original version of the resource and the current board version, including any updates made to the current board version by plugins or the admin. A third pane is presented below those two, with the suggested updates and resolution of conflicts. The UI presents a means of selecting, for each line in the top two panes, between the left and the right pane. Where conflicts require manual resolution, that selection is disabled, and in the third pane below, the usual style to indicate that manual resolution is required would be displayed:

```
    <<<<<<
    Line from newly released version
    ======
    Line from current board theme
    >>>>>>
```

    Potentially, more sophisticated means of automated resolution of otherwise manual conflicts could be developed, but probably not for the initial release of 1.9.

15. Regarding feature 1.2 in the list above (permanent modifications of resources by plugins), initially, a simple function corresponding to `find_replace_templatesets()` is to be implemented initially, named something like `find_replace_twig_tpls()`, which does effectively the same job, perhaps with better indication of errors such as changes failing to apply. If/when feature 3 (fine-grained modification functions) is later implemented, this function is to be deprecated.

16. Colour schemes. These could be implemented via [CSS variables](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties). Probably wouldn't be included in the initial 1.9 release, depending on demand / developer availability.

17. Macros (per feature number 15, the resource tracking of feature number 2, and the compatibility flagging of feature 9):

    17.1. Re feature number 2, macros are to be saved in the form of: `{ extension, extension version, serialized instructions, diff of changes for redundancy }`.

    17.2. In respect of feature number 15, macros allow the saving and (re)applying of sets of pre-defined custom changes, for advanced administrative usage. For example, to change the location of a custom widget:

       a. Install the plugin.

       b. Revert the plugin's changes to the selected theme using advanced options (resulting in an "attention needed" status for the theme).

       c. Create a custom "macro" with the desired modifications (programmatically / using the ACP / "recording" it and reviewing the end result).

       d. Mark the "macro" as resolving a particular change request (the theme system may suggest possible options, since it knows which plugin's changes are not currently applied).

       e. Apply the new "macro" to the theme (resulting in the "attention needed" status removed)

       [To decide:] Re (d) above: Should it be permissible for a macro to resolve multiple change requests? This would help with the manual resolution of conflicts between change requests of different plugins.

       [To decide:] Where to store such macros. Perhaps `inc/themes/macros`?

18. Feature number 1.2, permanent modifications (by plugins), is to be implemented by `extend.php` files, which return a structure as per macros above. Plugin updates which change this structure would be implemented as exemplified thus:

    18.1. Original theme version v1 exists, and a board theme inherits from it, with a plugin installed.

    18.2. New theme version v2 is uploaded.

    18.3. New plugin version is uploaded, which makes it compatible with theme version v2.

    18.4. The board theme's parent is switched to new version v2 (possibly resulting in warnings/conflicts as expected).

    18.5. The system reverts/discards the plugin's v1 edits applied in the Board theme.

    18.6. The system applies the plugin's changes appropriate for the v2 theme version in the board theme.

## Sources

Aside from ideas original to its authors, this document has been compiled from and inspired by several sources, sometimes quoting verbatim from them:

- The Community Forums thread [The 1.9 theme and template system](https://community.mybb.com/thread-232217.html).

- GitHub issue #3689 [1.9 Theme System](https://github.com/mybb/mybb/issues/3689), created by @Devilshakerz.

- Three Discord discussions, one [here](https://discord.com/channels/215876847634743296/378987603325747201/900882709839220736) in the MyBB server's public 1x-development channel, and two [here](https://discord.com/channels/215876847634743296/386813535176622080/905740893468913734) and [here](https://discord.com/channels/215876847634743296/386813535176622080/906869987543764992) in the staff-only 1x-development channel.
