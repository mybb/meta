# MyBB 1.9 Theme System
This document presents for discussion the general design, features, and roadmap of MyBB 1.9's theme system.

## Objectives
**Primary Objectives**
- change the template system to use Twig

**Secondary Objectives**
- change the storage method of theme data to files
- add support for theme metadata (e.g. author, version, license)
- improve theme management experience

**Additional Objectives**
- prepare the system for the support of automated installation of extensions

## Overview

### Theme System

#### Packages, and Data
The content and metadata of Resources, and most of Theme-related information, will be offloaded to the filesystem according to the principle of separating code and data:

- **Extension Package** → Filesystem

  Source files — including Twig templates, stylesheets, and static assets — will be stored under as files with the appropriate extension. That will include Resource Properties, like pages a stylesheet is attached to, stored in JSON files. Packages will be easily transferable between MyBB installations.

- **Runtime Data** → Database Tables

  Activated Themes will be tracked in the database table `mybb_themes`, and associated with a Package in the filesystem. These records will contain information specific to the individual forum installation, like permissions, customized title, or chosen color scheme, and will be referenced in other tables.

#### Themelets
Features related to Themes have a significant overlap with Plugins, which can supply their own templates or stylesheets. The new system introduces a logical structure — common to both types — grouping Resources introduced by a single Extension, together with their metadata, while allowing them to reside in the source Package's directory:

```
images/
scripts/
styles/
templates/
resources.json
```

The Extension System will process Resources contained in each active Extension's directory according to desired inheritance.

Themelets are the foundation of Themes and interfaced Plugins, which can use the same location to provide additional theming files and directories interpreted by either the Theme or Plugin System.

#### Version Data Retention
Past versions of Themelets supplied by activated Extensions will be retained on update, enabling improved compatibility, comparison, and conflict resolution features.

For example, after installing a Theme with version 1.0 and then uploading new versions 1.1 and 1.2, the templates, stylesheets, and other data for all three versions would be retained in the filesystem, and thus available for comparison — manual or automated — showing what changes need to be applied in child Themes.

#### Front-End Asset Processing
The system will support the SCSS format to compile browser-ready stylesheet files, and extend resource management features to support JavaScript files.

#### Partial Backward Compatibility with MyBB ≤ 1.8 Plugins
The Extension System will continue to support adding and attaching custom `eval()`-based templates and stylesheets to Themes, but may issue deprecation notices for Plugins using this method or Themes that have legacy Resources attached.

Interaction with existing Theme Templates (such as modifying them to add custom variables) using MyBB ≤ 1.8 functions and PHP syntax will not be supported, and plugins will need adjustment to modify Twig templates and use equivalent Twig syntax instead.

### Themes

#### Theme Types
The Extension System will internally use an additional type for third-party Themes. In addition to the *Core Theme* that cannot be modified (equivalent to 1.8's *MyBB Master Style*), distributed themes marked as *Original Themes* will be protected from accidental modifications.

The remaining Themes — *Board Themes* — will function as the customization layer, and will be editable by Plugins and ACP tools.

#### Theme Packages
Theme files will be placed in `inc/themes/<package-name>/` directories referencing codenames or local identifiers, and implement the Themelet structure, additionally including a `properties.json` file with additional configuration and presets, and `manifest.json` with author information and metadata.

#### Theme Activation
Administrators will able activate Themes by selecting one of the Packages in the filesystem. Activated Themes will store values for supported configuration options defined in Packages (like color scheme choices).

#### Update Experience Improvements
The existing mechanism of propagating Template updates will be extended to cover other Resource types, custom Themes, and Plugins, with improved usability. The system will show a comparison of upstream changes in inherited Resources that need to be copied manually, or will suggest an automatic patch.

### Plugins

#### Plugin Interfaces
Plugins will use the universal Themelet structure under `interface/` to supply own Interface Resources. This structure will additionally contain modification instructions for individual Themes, and a `manifest.json` file.

Custom Resources will be added at the beginning of the inheritance chain, meaning that Themes will be able to override them to adjust Plugins visually.

#### Interface Macros
The Extension System will expose a Theme modification API, and accept modification instructions known as *Macros*. These instructions will be handled by the System to apply, revert, and track theme changes requested by individual plugins.
The feature can be additionally extended to support creating custom Macros using the ACP.

##### Macro Tracking
*Macros* build on, and replace the existing `find_replace_templatesets()` and similar functions provided by third-party libraries. Compared to functions used in MyBB ­≤ 1.8 Plugins, however, the code dedicated to adjusting Themes will be limited to supplying the necessary additions or removals, and the responsibility for applying and reverting them will shift from custom code to the Extension System, which will store Macros and track their status for each Theme.

The scheme will allow a more fine-grained control for interactions between Plugins and Themes (like removing a Plugin's changes to an individual Theme, or substituting a Plugin's Macro with a custom one defined in the ACP).

Storing individual Macros will allow changes to be reverted even when the extension that introduced them is removed.

##### Macro Types
Changes that result in Theme resources being modified are defined Static Macros. This feature can be extended to allow Dynamic Macros, where changes are applied just before the Resources are rendered, leaving all source Themes intact.

## Design Notes
- ### Addressing MyBB ≤ 1.8 Theme Management Problems
  In MyBB ≤ 1.8, no established mechanism exists for managing theme updates and modifications. A forum administrator's workflow of installing third-party themes, plugins, and custom visual changes may be limited to importing a chosen theme and having its content modified directly, either by installing plugins or editing templates and stylesheets in the ACP. This results in a number of problems — often not evident at first — related to updating of imported themes.

  When importing a chosen theme again — in order to use its new version — the following usually apply:
  - plugins' changes have to be re-applied (usually by reactivating or reinstalling plugins, which may result in loss of data, and other side-effects),
  - manual changes have to be re-applied, and
  - manual changes have to be reviewed (to resolve potential conflicts).
  
  Experienced administrators may partially mitigate the problems by using existing inheritance features, in which case their workflow would rely on:

  - creating child themes — inheriting resources from imported themes — that contain custom edits, and
  - upgrading themes by importing their new versions, and switching child themes with custom edits to inherit from the newly imported version of their parent theme.
  
  It is, however, still the administrators' responsibility to remember to only apply changes in child themes, and to review changes that occurred in imported themes and apply them as needed to resources overwritten by child themes.

  MyBB ≤ 1.8's _Find Updated Templates_ functionality only supports reviewing changes to the _MyBB Master Style_ that need to be propagated manually, primarily because themes can only indicate compatibility with the core (referencing a MyBB version code; theme's templates are surfaced by the feature when their core version number is lower than that of the corresponding Master Style templates, and the template contents differ).

  <br>

  The new Theme types adapt the second approach, by marking imported Themes as *Original Themes*, and child Themes with forum-specific adjustments as *Board Themes*. This distinction allows to skip, disallow, or discourage automatic and manual edits to downloaded Themes, keeping their original content intact. This original data can then be used to revert any custom changes, and to compare changes between various versions of the same Theme.

  The Interface Macro feature separates the status of Theme changes from the status of Plugins (installation and activation), and introduces the possibility of saving manual changes with metadata, both of which could be inspected and toggled individually for each Theme.

- ### Theme Runtime Information
  Even though Theme Resources will be offloaded to the filesystem, some references for activated Themes will be stored in the database's `themes` table. These include instance-specific information that would not be applicable when moving a theme to another MyBB installation.
  
  <br>
  
  **Table:  Changes to the `mybb_themes` Database Table**
  
  Column | Changes | Notes
  --:|:--:|--
  `tid` | - | The internal ID of the activated Theme
  `package` | Added | The directory name of the Theme Package
  `version` | Added | The version of Theme Package's Resources to use
  `name` | Removed | Renamed to `title` to avoid confusion with Composer schema's `name`, and Extension codenames
  `title` | Added | The Theme's human-friendly display name
  `pid` | Removed | The ID of the theme's parent; may be discarded given that the parent Theme can already be determined via one of the JSON files in the filesystem
  `def` | Removed | Whether or not this is the default theme; the existing `default_theme` datacache will be used instead
  `properties` | - | Runtime settings (e.g. color preset choices); design-related properties moved to the `properties.json` file
  `stylesheets` | Deprecated | Data stored in `resources.json` file; kept for 1.8 plugin compatibility
  `allowedgroups` | - | Instance-specific references to user groups; may later be normalized using a pivot table
  
  <br>
  
  The remaining  database tables specific to Theme Resources (`templategroups`, `templates`, `templatesets`, `themestylesheets`) will be deleted when support for `eval()`-based templates is removed.
  
- ### Inheritance
  During the rendering process, individual Interface Resources will be queried from available sources (the Core, active Plugins, selected Theme and its parent Themes) in accordance with established overriding rules and priority determined by the inheritance hierarchy.
  
  The resulting Resolved Resources will be associated with individual final Themes (that can be selected by users, or as the application's default) and may be compiled and cached.
  
  The list of source Themes will be built recursively, starting with the requested final Theme, by prepending declared parent Themes to the chain. Inheritance information declared lower in the chain will have priority, allowing child Themes to override declarations of parent Themes and their versions.

  <br>
  
  **Diagram: Interface Resource Inheritance Overview**
  
  ![interface-resource-inheritance-overview](https://user-images.githubusercontent.com/8020837/143784234-2915089c-6f8c-48a1-ad83-e8e57f75dee0.png)
  
  <br>
  
  **Table: Themelets' Ability to Override Extension Resources**
  
  Acting Extension Type \ Target Type | Core Theme | Plugin | Original Theme | Board Theme
  :--:|:--:|:--:|:--:|:--:
  **Core Theme** | ❌ No | ❌ No | ❌ No | ❌ No
  **Plugin** | ❌ No | ❌ No / ➕ Add only / ✅ Yes (depending on chosen approach) | ❌ No | ❌ No
  **Original Theme** | ✅ Yes | ✅ Yes | ✅ Yes (parents only) | ❌ No
  **Board Theme** | ✅ Yes | ✅ Yes | ✅ Yes (parents only) | ✅ Yes (parents only)
  
  - A **Core Theme** is the first source in the inheritance chain, and therefore doesn't inherit Resources from other sources, and cannot override any other source.
  - A **Plugin Themelet** may only supply own Resources that are added to the inheritance chain, or add Resources to Themelets of other Plugins'.
  - An **Original Theme** may inherit and override Resources from the Core Theme, parent Original Themes, and Plugin Themelets.
  - A **Board Theme** may inherit and override Resources from Plugin Themelets and all parent Themes.
  
  <br>
  
  **Table: Extensions' Ability to Affect Resources**
  
  Acting Extension and Method \ Target | A Theme's Themelet | A Theme's Resources in a Plugin's Namespace | A Plugin's Themelet | A Plugin's Resources in another Plugin's Namespace | A Plugin's Function/Macro
  --|:--:|:--:|:--:|:--:|:--:
  Theme — through inheritance | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No
  Plugin — through Inheritance | ❌ No | ❌ No | ❌ No / ➕ Add only / ✅ Yes (depending on chosen approach) | ❌ No / ➕ Add only / ✅ Yes (depending on chosen approach) | ❌ No
  Plugin — through Function/Macro | ✅ Yes | ✅ Yes | ❌ No | ❌ No / ✅ Yes (depending on chosen approach) | ❌ No

  - **Themes**:
    - can override Resources of Plugins and parent Themes using inheritance,
    - cannot supply their own Macros.
  - **Plugins**:
    - cannot override Theme Resources using inheritance (Plugins' Resources are part of the inheritance base),
    - can use Functions/Macros to modify Board Themes.
  - **Function/Macro** definitions cannot be influenced externally.
  
- ### Resource Versioning
  Resource inheritance and dependencies between Extensions will rely on references similar to those used by software package managers, allowing Extension developers and administrators to lock them down to specific versions for stability purposes. These references can link:
  
  - runtime implementations with Packages (e.g. the forum default Theme pointing to a Package directory in the filesystem — using values of the `package` and `version` columns in the `mybb_themes` table),
  - Packages with other Packages (e.g. a Theme Package referencing a parent Theme Package to inherit Resources from — using names and versions stored in JSON files).
  
  To facilitate this behavior, supplied Themelets will be associated with versions of Extensions they were originally included in, and stored indefinitely in the filesystem. Notably, only a Themelet — with Resources that can be interpreted by the Theme System — will be preserved, rather than the complete Extension Package.
  
  The lifecycle of version-specific Themelets is loosely related to their status:
  
  1. **latest** — the Themelet version is associated with the latest version of an Extension,
  2. **active** — the Themelet version is outdated, but is configured as the main Themelet for an active Extension, or is inherited from by another active Extension,
  3. **unused** — the Themelet version is outdated and is not referenced in any inheritance chain, but may continue to be used for comparison and auditing purposes,
  4. **purged** — the Themelet version is deleted.
  
  This lifetime is not expected to be continuous, as arbitrary versions can be referenced throughout the application at any time (e.g. by Extensions or through configuration in the ACP).
  
  Old Themelet versions can be removed manually from the filesystem, and additional tooling may be provided in the ACP for reviewing and purging unused versions.

- ### Versioned Themelet Storage
  Given that Themelets are subsets of Extension Package files, and all versions — including the most recent ones — are expected to have nearly equal status (as any version can be referenced throughout the application at any time), their storage structure would be similar to:
  
  _Extensions directory_ (`inc/plugins/`, `inc/themes/`) ← _Package_ (e.g. `<codename>/`) ← _Package Themelets_ (e.g. `interface/`) ← _Themelet_ (e.g. `<version>/`)

  Interactions with this structure affect:
  - **System Complexity**

    A logical directory structure is preferred to simplify the implementation and related code (e.g. querying data using normalized paths), and to minimize cognitive load.
  - **User Experience and Data Integrity**

    Common operations, like uploading or updating Extensions, should be simple and have predictable effects. The mechanism should assure that no data is overridden accidentally.
  - **Developer Experience**

    While Extension versions will have to be declared in manifest files and changed on subsequent releases, the potential practice of renaming directories — which may be nested deeper than the Extension Package's main directory — may be unexpected, forgotten, or lead to practical problems.
    
    A common development workflows may involve:
    1. copying the Extension's files into a MyBB installation (e.g. from a code repository),
    2. editing the Extension's source code and observing the effects,
    3. comparing and copying files from the MyBB installation back into the source (e.g. a code repository)
  
    in which case static directory names would be preferred to preserve code history and allow quick comparison.

  Specifically, the following desired features can be identified:
  - **Normalized Paths** (System Complexity)

    The storage structure follows and illustrates the system's logical hierarchy.
  - **Overwrite Protection** (User Experience and Data Integrity)

    New versions of the Extension can be uploaded without overwriting past versions' data.

  - **Recognition of Deleted Files** (User Experience and Data Integrity)

    Removed files, present in past versions, won't be incorrectly included again after uploading the new version.
  - **Importing with Static Paths** (Developer Experience)

      The Extension's files can be added to the installation without changing directory names to indicate specific versions.
  - **Live Editing with Static Paths** (Developer Experience)

      The same file paths as the ones uploaded can be used to edit and preview changes.
  - **Exporting with Static Paths** (Developer Experience)

      The Extension's files can be downloaded to the same directory structure without adjusting versioned directory names.
  - **Copy Synchronization** (Developer Experience)

    If copies of the Extension's files are made (e.g. to allow safe overwriting), changes are propagated to the copies properly.
  
  <br>
  
  The following types of versioning methods were considered:
  
  - **Passive**, in which the directories are versioned before being used by the Extension System:
    - **Manual** — the simplest approach that puts the burden of naming Themelet directories according to their version on Extension developers
    - **On Detection** — an improved approach that allows developers to distribute packages with static directory names, which are automatically renamed according to the Extension's version
  - **Active**, in which the target location with versioned Themelets is managed exclusively by the application, and Extension files are uploaded through the ACP or to a staging directory and processed by the application:
    - **Processing (Simple)** — Extension files are uploaded to the ACP or a temporary location and processed to rename directories according to the Extension's version
    - **Processing (Compatible)** — an improved approach that stores the latest Themelet version in a directory with a static name, allowing developers to access it for editing and exporting more easily
  - **Mixed**, in which administrators and developers can interact with the target location, but some intervention by the application is required:
    - **On Archiving** — the most recent Themelet is kept in a directory with a static name, and is renamed to `<version>/` through user action before a new version is uploaded
    - **Cold Duplicate** — Extension files are duplicated on detection to a `<version>/` directory; the static-named directory overrides content stored in `<version>/` directories as determined by the version specified in the manifest file; the files may be synchronized continuously when in development mode or on request
    - **Hot Duplicate** — an improved approach, where Extension files from the static-named directory are synchronized to the `<version>/` copy, where data is accessed from; may be I/O-intensive, and would require user action to break synchronization before uploading new versions
  
  <br>
  
  **Table: Themelet Versioning Methods**
  
  Type | Versioning Method | Runtime Source Directory | Overwrite Protection | Recognition of Deleted Files | Importing with Static Paths | Live Editing with Static Paths | Copy Synchronization | Exporting with Static Paths 
  --|--|:--:|:--:|:--:|:--:|:--:|:--:|:--:
  **Passive** | Manual | `<version>/` | ✅ Yes | ✅ Yes | ❌ No | ❌ No | n/a | ❌ No 
  **Passive** | On Detection | `<version>/` | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No | n/a | ❌ No 
  **Active** | Processing (Simple) | `<version>/` | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No | n/a | ❌ No 
  **Active** | Processing (Compatible) | static path (for latest version) or  `<version>/` (for older versions) | ✅ Yes | ✅ Yes | ✅ Yes | ⚠️ Yes; different location | n/a | ⚠️ Yes; different location 
  **Mixed** | On Archiving | static path (for latest version) or  `<version>/` (for older versions) | ⚠️ Yes; action required | ✅ Yes | ✅ Yes | ✅ Yes | n/a | ✅ Yes 
  **Mixed** | Cold Duplicate | static path (priority) or `<version>/` | ✅ Yes | ❌ No | ✅ Yes | ✅ Yes | ⚠️ Yes; action required| ✅ Yes 
  **Mixed** | Hot Duplicate | `<version>/` | ⚠️ Yes; action required | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes 
  
  <br>
  
- ### Resource Namespaces
  Resources from Themes and Plugins are saved and accessed using a single namespace, i.e. without any logical separation, which may lead to overlapping Resource paths.
  
  This results in Extensions "polluting" the general-use namespace, which may lead to accidental collisions of Resource paths (Theme–Plugin, Plugin–Plugin), e.g. when using common names in template groups. Even when Plugins follow best practices to contain own Resources within unique paths using codenames (e.g. `templates/<codename>/`), Theme–Plugin conflicts may occur when valid Extension codenames use common words that may be used in Themes' Resource paths.
  
  For example, when `common_name` is used simultaneously as:
  
  - a template group name used by a Theme,
  - a template group name used by Plugin A (which doesn't use unique paths to contain own Resources), and
  - a codename used by Plugin B (which follows best practices and uses its codename to contain own Resources),
  
  the Resource paths provided by the three Extensions may collide:
  
  ```twig
  {# referencing Theme templates #}
  {% include 'template.twig' %}
  {% include 'common_name/template.twig' %}
   
  {# referencing Plugin A's templates #}
  {% include 'template.twig' %}
  {% include 'common_name/template.twig' %}
   
  {# referencing Plugin B's templates #}
  {% include 'common_name/template.twig' %}
  ```
  
  which would result in:
  - potentially unexpected behavior (when a Plugin's Resources are accidentally overridden by a Theme, given that Themes always override Resources from the inheritance base), and/or
  - undefined behavior (when a Plugin's Resources are accidentally overridden by another Plugin, given that Plugins don't have any explicit hierarchy or order).
  
  #### Separate Namespaces for Plugins
  To address this problem, Resources supplied by Plugin Themelets can be placed in — and accessed from — their own namespace, which would result in distinct and unique paths managed by the Extension System.
  
  In that case, no accidental Theme–Plugin or Plugin–Plugin Resource path collisions would occur:
  
  ```twig
  {# referencing Theme templates #}
  {% include 'template.twig' %}
  {% include 'common_name/template.twig' %}
  
  {# referencing Plugin A's templates #}
  {% include '@plugin_a/template.twig' %}
  {% include '@plugin_a/common_name/template.twig' %}
   
  {# referencing Plugin B's templates #}
  {% include '@plugin_b/common_name/template.twig' %}
  ```
  
  Additionally, the design can help prevent Plugins from overwriting Resources in the main namespace without doing so explicitly using Interface Macros (or other Theme modification functions).
  
  #### Populating Plugin Namespaces
  Contributions from Themes to a Plugin's namespace (intended to override the Plugin's original Resources) can be made possible by combining Resources placed in directories referencing the target Extension's codename with its own Themelet.
  
  These can be implemented as e.g.:
  
  - **Sub-directories in Special Format**

    Resources intended for a namespace of a particular Extension are placed in a special directory `ext-<codename>` (e.g. `templates/ext-plugin-name/`).
  - **Separate Directories for Extensions**

    Resources intended for a namespace of a particular Extension are placed in `extensions/<codename>/<version>/` (e.g. `extensions/plugin-name/v1/templates/`).
  
    This unifies the structure designed for Interface Macro files (`macros/<codename>/<version>/`) with Resources to be added to the Extension's namespace.
  
  If desired, Plugins may be similarly permitted to contribute to other Plugins' namespaces.
  
  #### Source Priority in Plugin Namespaces
  In addition to preventing accidental collisions, namespaces allow for better control of intentional overrides (compared to combining all Resources in a single namespace in undefined order).
  
  While Themes are expected to override Plugin Themelets (by assigning higher priority to Themes' contributions), namespaces allow for better control of external Plugins' contributions, leaving the possibility of:
  - disallowing Plugin–Plugin contributions completely (by not recognizing external Plugins' sources),
  - only allowing non-conflicting contributions (i.e. only new Resources) to another Plugin's namespace (by assigning higher priority to original Plugin's sources), or
  - allowing any contributions, including the overriding of original Resources, while ensuring that external overrides work correctly (by assigning higher priority to external Plugins' contributions).
  
  In the last two cases, contributions from two or more third-party Extensions to a single Plugin's namespace may result in collisions, given the lack of priority-defining hierarchy (which exists for Themes). This may be addressed by allowing Plugin developers or administrators to set explicit priority for Plugins or individual Resources, whether absolute (similar to numeric forum order) or relative (referencing conflicting, sibling extensions in "before" and "after" lists).
  
  <br>
  
  **Table: Handling of Resource Conflicts by Namespaces Approach**
  
  Events \ Approach | Single Namespace | Separate Namespaces
  --:|:--|:--
  Accidental Theme-Plugin collisions | ✅ Yes | ❌ No
  Accidental Plugin-Plugin collisions | ✅ Yes | ❌ No
  Intentional overrides of a Plugin's Resources by another Plugin | ⚠️ Yes; undefined priority | 🚦 Yes; controlled priority
  Intentional overrides of a Plugin's Resources by multiple other Plugins | ⚠️ Yes; undefined priority of overriding Plugins | ⚠️ Yes; undefined priority of overriding Plugins
  
  <br>
  
  **Table: Themelet Namespace Interactions by Extension Type**
  
  Origin Extension Type | Themelet Destination Namespace | Source in Main Namespace | Source in Own Extension Namespace | Source in Any Extension Namespace
  :--:|:--:|:--:|:--:|:--:
  **Core Theme** | main | ✅ Yes | no own namespace | ❌ No
  **Plugin** | extension's own | ❌ No | ✅ Yes (highest/lowest priority, depending on chosen approach) | ❌ No / ➕ Add only / ✅ Yes (depending on chosen approach)
  **Original Theme** | main | ✅ Yes (overriding parents) | no own namespace | ✅ Yes (overriding parents)
  **Board Theme** | main | ✅ Yes (overriding parents) | no own namespace | ✅ Yes (overriding parents)
  
  <br>
  
  **Diagram: Example Registration of Interface Resources in Namespaces**
  
  ![registration-of-interface-resources-in-namespaces](https://user-images.githubusercontent.com/8020837/143784207-03067ee6-cbac-4b41-a6ed-130c46d2f4e1.png)
  
  <br>
  
  #### Namespaces in Themes
  It's possible to take advantage of namespaces in Themes to separate Resources intended for different interface types, such as:
  - the forum front-end,
  - ACP,
  - parser output,
  - email content, or
  - error messages.
  
  This separation may help with loading such modules individually (e.g. in the ACP, the forum front-end templates are not needed).
  
- ### Directory Structure
  The following storage choices were considered for each logical group in the hierarchy of Resources:
  
  1. Extension Type
     - A. Themelets stored in the Extension Directories (i.e. split between `inc/themes/` and `inc/plugins/`, according to type)
     - B. Themelets stored in the same directory (e.g. `inc/themes/`, splitting the Extension's Themelet from other files)
  1. Extension's Target Interface Type/Namespace
     - A. Extension Directories spread across the filesystem according to type (e.g. `inc/themes/`, `admin/themes/`)
     - B. Extension Directories stored in subdirectories according to type (e.g. `inc/themes/frontend/`, `inc/themes/admin/`)
     - C. Extension Directories stored in the same directory for all types (e.g. `inc/themes/`)
  1. Extension Package
     - A. Extensions stored in directories referencing the package name (`<package-name>/`)
  1. Themelet Path
     - A. Themelets stored in `interface/` subdirectories of Extension Directories
     - B. Themelets of Plugins stored in `interface/` subdirectories of Extension Directories, and Themelets of Themes directly in Extension Directories
     - C. Themelets of all Extensions stored directly in Extension Directories
  1. Latest Themelet Version (depending on chosen approach)
     - A. latest Themelets stored in subdirectories of Extension Themelet Paths (e.g. `interface/current/`)
     - B. latest Themelets stored directly in Extension Themelet Paths (e.g. `interface/`)
  1. Old Themelet Versions (depending on chosen approach)
     - A. old Themelets stored in an external structure (e.g. `inc/themelets/<package-name>/<version>/`)
     - B. old Themelets stored in subdirectories of Extension Themelet Paths (e.g. `interface/<version>/`)
  1. Destination Namespace (depending on chosen approach)
     - A. all Themelets stored in directories named according to type or namespace (e.g. `interface/base/` for main resources, and `interface/<namespace>/` for specific namespaces)
     - B. namespace-specific Themelets stored in a directory for non-main namespaces (e.g. `interface/extensions/<namespace>/`)
     - C. namespace-specific Themelets stored in main Themelet directories (e.g. `interface/ext-<namespace>/`)
     - D. namespace-specific Themelets merged with main Themelets (e.g. `interface/<resource-type>/ext-<namespace>/`
  1. Third-Party Extension Version (depending on chosen approach)
     - A. version-specific Themelets stored in directories referencing the target Extension version (`<extension-version>/`)
  
- ### JSON Data Files
  - #### `manifest.json`
    Defines metadata associated with an Extension. Expected to be compatible with the [composer.json schema](https://getcomposer.org/doc/04-schema.md#json-schema).

    Contains:
    - **codename** (`codename`)
    - **title** (`extra.title`), a human-friendly name displayed in the UI
    - **description** (`description`)
    - **version** (`version`), compatible with PHP's [`version_compare()`](https://www.php.net/manual/en/function.version-compare.php)
    - **authors** (`authors`), as defined by the Composer schema
    
    The `name` property is reserved for package names used in the Composer/Packagist ecosystem.
    
  - #### `properties.json`
    Defines Theme-related settings and presets.
  
    Contains:
    - inheritance information
    - Color Sets
    
  - #### `resources.json`
    Defines data associated with Resources, addressable by type and path.
  
    Contains:
    - acknowledgements of parent versions (marking Resources as compatible)
    - pages to which a stylesheet is attached

- ### Resource Groups
  Themelet Resource type directories (e.g. `templates/`, `styles/`) may contain subdirectories separating Resources into groups (shown in a expandable/collapsible hierarchy in the ACP), which may be nested.

  The system may support translatable titles and descriptions for Resources and groups, provided in the JSON data files.

- ### Naming
  - **Theme Packages**
    
    Extension Package directories, in addition to uniquely identifying packages, will be used to determine their origin:
    - system (using reserved static names, e.g. `core` or `core.default`)
    - instance-limited packages (using reserved, codename-incompatible format, referencing an internal ID, e.g. `theme.<id>` for Board Themes)
    - distributable packages (using the codename format enforced by the Extend platform: `[a-z_]+`)
    
  - **Resource Directories**

    Directories grouping Themelet Resources by type are named according to their purpose, rather than specific language or technology.
    E.g. CSS files are placed in `styles/`, instead of `css/` or `stylesheets/`, as the directory may also contain SCSS files.

- ### Resource Modification Functions
  Initially, a simple function corresponding to `find_replace_templatesets()` may be implemented to allow operations on Twig templates, possibly with better indication of errors (e.g. changes failing to apply).

  With the implementation of Interface Macros, such functions will be deprecated.

- ### Legacy Resources
  In order to support Plugins' Resources inserted into the legacy, database-only system, the existing code will be used to combine them with Resources queried from the new system during resolution.

  Similarly, Resources from both systems will be combined in Resource management views in the ACP, and the UI may indicate the deprecation status for those associated with the legacy system.

  <br>

  **Table: Support of Legacy Resources in MyBB 1.9**
  Resource Type | Global Set | Theme Set
  --:|:--:|:--:
  Custom PHP Templates in Database | ⚠️ Yes, deprecated | ⚠️ Yes, deprecated
  Custom Stylesheets in Database | n/a | ⚠️ Yes, deprecated

  <br>

  Plugins will continue to be able to perform insert, removal, and modification operations on such Resource types programmatically, using existing functions and libraries. These types will not be supported for Themes, which can supply no executable code recognized by the system.

## Examples
- #### `resources.json` Structure
  ```json
  {
      "templates": {
          "group/name.twig": {
              "last_modified": 123456789,
              "parents": [
                  "core@v1.9.1",
                  "original_theme@v2",
                  "theme.1@123456789"
              ]
          }
      }
  }
  ```

- #### Template Source Directory List Population
  A function adding template paths of the core and extensions to Twig's Filesystem Loader. Assumes usage of separate namespaces for Extensions.
  
  ```php
  function addTwigTemplatePaths(\Twig\Loader\FilesystemLoader $loader): void
  {
      // add sources, starting at the beginning of the inheritance chain
  
      // add core theme's templates
      $loader->prependPath(
          path: $coreTheme->getTemplatePath(),
      );
  
      // add plugins' templates
      foreach ($activePlugins as $plugin) {
          // add plugin's templates to its own namespace
          $loader->prependPath(
              path: $plugin->getTemplatePath(),
              namespace: $plugin->getCodename(),
          );
  
          // add other extensions' templates for the plugin's namespace
          foreach ($extensions as $extension) {
              if (
                  $extension->getType() !== ExtensionSystem::THEME_TYPE_CORE &&
                  $extension->getCodename() !== $plugin->getCodename()
              ) {
                  $loader->prependPath(
                      path: $plugin->getExtensionTemplatePath($plugin->getCodename()),
                      namespace: $plugin->getCodename(),
                  );
              }
          }
      }
  
      // add theme ancestors' templates (from furthest to closest)
      foreach ($theme->getAncestors() as $theme) {
          if ($theme->getType() !== ExtensionSystem::THEME_TYPE_CORE) {
              $loader->prependPath(
                  path: $theme->getTemplatePath(),
              );
          }
      }
  
      // add theme's own templates
      $loader->prependPath(
          path: $theme->getTemplatePath(),
      );
  }
  ```

- #### Interface Macro Definition
  A Macro can be expressed programmatically as data objects in a form similar to:

  ```php
  // an array of theme changes
  return [
      // add a template
      \MyBB\ExtendTheme\Template('custom_page.twig')
          ->setContent('Hello {{ username }}'),
  
      // modify existing template
      \MyBB\ExtendTheme\Template('index.twig')
          ->insertBefore(
              '<a href="custom_page.php">Custom Page</a>',
              '<!-- end: navigation -->',
          ),
  
      // add a stylesheet from file
      \MyBB\ExtendTheme\Stylesheet('custom_page.css')
         ->setContentFromFile(MYPLUGIN_DIRECTORY . '/stylesheets/custom_page.css')
         ->setAttachedTo('custom_page.php'),
  ];
  ```


- #### Resource Conflict Resolution
  The `Text_Diff` third-party library already in use for MyBB 1.8 could be leveraged, in particular via its support for three-way diffs. It seems that it would need some modifications, but, generally, it could support the following implementation:

  > Two panes are presented side by side. The left pane shows the differences for each line between the original version of the resource and the newly released version of the resource (using the same formatting and styling as the current diffs for 1.8). The right pane shows the differences for each line between the original version of the resource and the current board version, including any updates made to the current board version by plugins or the admin. A third pane is presented below those two, with the suggested updates and resolution of conflicts. The UI presents a means of selecting, for each line in the top two panes, between the left and the right pane. Where conflicts require manual resolution, that selection is disabled, and in the third pane below, the usual style to indicate that manual resolution is required would be displayed:

  ```diff
  <<<<<<
  Line from newly released version
  ======
  Line from current board theme
  >>>>>>
  ```

  More sophisticated means of automated resolution of otherwise manual conflicts could be developed.

## General Roadmap
1. Fundamentals
   - move & add the default Theme's Resources according to the new filesystem structure
1. Theme System Basics
   - copying static files (CSS, images) to web-accessible directories
   - SCSS support
   - basic Theme Resource modification functions
   - ACP Template & stylesheet editing
1. Theme System Improvements
   - Interface Macro API, Static Interface Macros
   - ACP UI for customizing values of CSS/SCSS variables
1. Advanced Theme System Features
   - Dynamic Interface Macros
   - Storing history of manual and automated changes to Resources
1. Extension System Improvements
   - `manifest.json` for Plugins
   - Ability to install Extensions using Composer

## Definitions
*general*

- **ACP**, **Admin CP** — the Admin Control Panel
- **Core** — the MyBB forum software in its unmodified state and without Extensions
- **Extend** — the official platform for MyBB Extensions hosted on MyBB.com
- **Front-end** — the publicly accessible forum interface part of the software
- **Web Root Directory** — MyBB's main directory (referred to by PHP constant `MYBB_ROOT`)

*extensions*

- **Extension** — a third-party component recognized by the application
  - **Plugin** — a component intended to modify the application's behavior
  - **Theme** — a component defining the user interface
    - **Core Theme** — a Theme belonging to the Core, defining MyBB's default user interface; not editable during normal use
    - **Original Theme** — a Theme maintained by third-party; not editable during normal use
    - **Board Theme** — an editable Theme
  - **Extension Codename** — a unique identifier associated with an extension, limited to a-z letters and underscore `_`, registered on the MyBB Extend platform
  - **Extension Package** — a distributable set of files associated with an Extension
  - **Extension Directory** — a directory containing files from an Extension Package, located directly under `inc/plugins/` or `inc/themes/`
  - **Extension Metadata** — information about the extension, such as the author and license, used in Extension Management
  - **Extension Manifest File** — a `manifest.json` file in JSON format, compatible with [Composer schema](https://getcomposer.org/doc/04-schema.md), containing Extension Metadata, located directly in an Extension Directory
- **Extension System** — the Theme System and the Plugin System
  - **Plugin System** — code responsible for execution of Plugins
  - **Theme System** — code responsible for execution and rendering of Themes

- **Extension Management** — code and events related to configuring Extensions

*interface*

- **Interface Resource** — an interface-related file intended to be processed or exposed through HTTP
  - **Template** — code interpreted by a template engine
    - **PHP Template** — a MyBB 1.8-style Template interpreted using `eval()` statements
    - **Twig Template** — a MyBB 1.9-style Template interpreted by Twig
  - **Front-end Asset** — content and files intended to be exposed though HTTP
- **Interface Resource Property** — data associated with an Interface Resource (e.g. pages to which a stylesheet is attached)

*theme system*

- **Themelet** — Interface Resources with associated properties [Interface Resource Properties]
  - - **Self-contained Themelet** — a Themelet with only custom Interface Resources
    - **Multi-namespace Themelet** — a Themelet with Interface Resources that may be custom, or override those of selected Extensions
  - **Direct Themelet** — a collection of a Themelet's Resources and associated Properties whose target namespace is managed by the system
  - **Namespace Themelet** — a collection of Themelet's Resources and associated Properties whose target namespace is defined explicitly
- **Interface Resource Properties File** — a `resources.json` file in JSON format, located directly in a Themelet Directory
- **Themelet Directory** — a directory containing Interface Resources separated into subdirectories named according to their type (e.g. `templates/`, `css/`, `js/`, etc.), and the Interface Resource Properties File
- **Themelet Inheritance Chain** — an ordered list of Extensions, according to which a final Themelet is resolved, where priority is defined by the number of steps away from the end of the chain
- **Themelet Inheritance Base** — Themelet of the Core Theme and Plugin Themelets of active Plugins
- **Resolved Themelet** — Themelet collected or composed according to a defined Themelet Inheritance Chain
- **Compiled Themelet** — Resolved Themelet prepared for output or final interpretation (e.g. Twig templates converted to PHP code) and web-accessible static files

*themes*

- **Theme Property** — data associated with a Theme (e.g. color presets)
- **Theme Properties File** — a `properties.json` file in JSON format containing Theme Properties, located directly in a Theme Definition Directory
- **Theme Definition** — a Themelet associated with a Theme, and Theme Properties
- **Theme Definition Directory** — a directory containing a Theme Definition, associated with a Theme and its version

*macros*

- **Interface Macro** — a set of changes to a Themelet that can be stored, applied, and reversed
  - **Static Interface Macro** — an Interface Macro that operates on a Themelet and causes it to be stored in modified state
  - **Dynamic Interface Macro** — an Interface Macro that operates on a Resolved Themelet and causes a Compiled Themelet to be stored in modified state

*plugin interface*

- **Plugin Interface Definition** — interface-related data supplied by a Plugin
  - **Plugin Themelet** — a Themelet supplied by a Plugin
  - **Plugin Interface Macro** — an Interface Macro supplied by a Plugin
- **Plugin Interface Macro Directory** — an innermost directory in the `<extension codename>/<extension version>/` structure associated with a specific Extension and its version located in a Plugin Interface Definition Directory, containing a Plugin Interface Macro File
- **Plugin Interface Macro File** — an `extend.php` file defining a Plugin Interface Macro, located in the Plugin Interface Macro Directory
- **Plugin Interface Definition Directory** — an innermost directory in the `interface/<plugin version>` structure located directly in a Plugin's Extension Directory, associated with a Plugin's version, additionally containing a `macros/` subdirectory with Plugin Interface Macro Directories

## References
- GitHub issue #3689 [1.9 Theme System](https://github.com/mybb/mybb/issues/3689)
- Community Forums thread [The 1.9 theme and template system](https://community.mybb.com/thread-232217.html)
- [Discord server](https://mybb.com/discord) discussions in `#1x-development`, `#staff-1x-organization` channels
