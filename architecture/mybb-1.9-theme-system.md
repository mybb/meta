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
Features related to Themes have a significant overlap with Plugins, which can supply their own templates or stylesheets. The new system introduces a logical structure — common to both types — grouping Resources introduced by a single Extension by namespace and type, together with their metadata:

```
frontend/
    images/
    scripts/
    styles/
    templates/
    resources.json
acp/
    ...
```

The Extension System will process Resources contained in each active Extension's directory according to desired inheritance.

Themelets are the foundation of Themes and interfaced Plugins, which can use the same location to provide additional theming files and directories interpreted by either the Theme or Plugin System.

#### Version Data Retention
Past versions of Themelets supplied by activated Extensions will be retained on update, enabling improved compatibility, comparison, and conflict resolution features.

For example, after installing a Theme with version 1.0 and then uploading new versions 1.1 and 1.2, the templates, stylesheets, and other data for all three versions would be retained in the filesystem, and thus available for comparison — manual or automated — showing what changes need to be applied in child Themes.

#### Asset Processing
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
Administrators will be able to activate Themes by selecting one of the Packages in the filesystem. Activated Themes will store values for supported configuration options defined in Packages (like color scheme choices).

#### Update Experience Improvements
The existing mechanism of propagating Template updates will be extended to cover other Resource types, custom Themes, and Plugins, with improved usability. The system will show a comparison of upstream changes in inherited Resources that need to be copied manually, or will suggest an automatic patch.

### Plugins

#### Plugin Interfaces
Plugins will use the universal Themelet structure under `interface/` to supply own Interface Resources, and contain modification instructions for individual Themes.

Custom Resources will be added at the beginning of the inheritance chain, meaning that Themes will be able to override them to adjust Plugins visually.

Similar to Themes, the system will use `manifest.json` files for metadata.

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
  
  Experienced administrators may partially mitigate these problems by using existing inheritance features, in which case their workflow would rely on:

  - creating child themes — inheriting resources from imported themes — that contain custom edits, and
  - upgrading themes by importing their new versions, and switching child themes with custom edits to inherit from the newly imported version of their parent theme.
  
  It is, however, still the administrators' responsibility to remember to only apply changes in child themes, and to review changes that occur in imported themes in order to propagate them, as needed, to resources overwritten in child themes.

  MyBB ≤ 1.8's _Find Updated Templates_ functionality only supports reviewing changes to the _MyBB Master Style_ that need to be propagated manually, primarily because themes can only indicate compatibility with the core (referencing a MyBB version code; a theme's templates are surfaced by the feature when their core version number is lower than that of the corresponding Master Style templates, and the template contents differ).

  <br>

  The new Theme types adapt the second approach, by marking imported Themes as *Original Themes*, and child Themes with forum-specific adjustments as *Board Themes*. This distinction allows to skip, disallow, or discourage automatic and manual edits to downloaded Themes, keeping their original content intact. This original data can then be used to revert any custom changes, and to compare changes between various versions of the same Theme.

  The Interface Macro feature separates the status of Theme changes from the status of Plugins (installation and activation), and introduces the possibility of saving manual changes with metadata, both of which could be inspected and toggled individually for each Theme.

- ### Theme Runtime Information
  Even though Theme Resources will be offloaded to the filesystem, some references for activated Themes will be stored in the database's `themes` table. These include instance-specific information that would not be applicable when moving a Theme to another MyBB installation.
  
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
  - A **Plugin** may only supply own Resources that are added to the inheritance chain, or add Resources to Themelets of other Plugins.
  - An **Original Theme** may inherit and override Resources from the Core Theme, parent Original Themes, and Plugin Themelets.
  - A **Board Theme** may inherit and override Resources from Plugin Themelets and all parent Themes.
  
  <br>
  
  **Table: Extensions' Ability to Affect Resources**
  
  Acting Extension and Method \ Target | A Theme's Themelet | A Theme's Resources in a Plugin's Namespace | A Plugin's Themelet | A Plugin's Resources in another Plugin's Namespace | A Plugin's Function/Macro
  --|:--:|:--:|:--:|:--:|:--:
  Theme — through inheritance | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No
  Plugin — through Inheritance | ❌ No | ❌ No | ❌ No / ➕ Add only / ✅ Yes (depending on chosen approach) | ❌ No / ➕ Add only / ✅ Yes (depending on chosen approach) | ❌ No
  Plugin — through Function/Macro | ✅ Yes | ✅ Yes | ❌ No / ✅ Yes (depending on chosen approach) | ❌ No / ✅ Yes (depending on chosen approach) | ❌ No

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
  
  The lifecycle of version-specific Themelets is loosely related to their usage status:
  
  1. **latest** — the Themelet version is associated with the latest version of an Extension,
  2. **active** — the Themelet version is outdated, but is configured as the main Themelet for an active Extension, or is inherited from by another active Extension,
  3. **unused** — the Themelet version is outdated and is not referenced in any inheritance chain, but may continue to be used for comparison and auditing purposes.
  
  This lifetime is not expected to be continuous, as arbitrary versions can be referenced throughout the application at any time (e.g. by Extensions or through configuration in the ACP).
  
  Old Themelet versions can be removed manually from the filesystem, and additional tooling may be provided in the ACP for reviewing and purging unused versions.

- ### Versioned Themelet Storage
  Given that Themelets are subsets of Extension Package files, and all versions — including the most recent ones — are expected to have nearly equal status (as any version can be referenced throughout the application at any time), their storage structure would be similar to:
  
  _Extensions directory_ (`inc/plugins/`, `inc/themes/`) ← _Package_ (e.g. `{codename}/`) ← _Package Themelets_ (e.g. `interface/`) ← _Themelet_ (e.g. `{version}/`)

  Interactions with this structure affect, in order of importance:
  - **User Experience and Data Integrity**

    Common operations, like uploading or updating Extensions, should be simple and have predictable effects. The mechanism should assure that no data is overridden accidentally.
  - **Developer Experience**

    While Extension versions will have to be declared in manifest files and changed on subsequent releases, the potential practice of renaming directories — which may be nested deeper than the Extension Package's main directory — may be unexpected, forgotten, or lead to practical problems.
    
    Common development workflows may involve:
    1. copying the Extension's files into a MyBB installation (e.g. from a code repository),
    2. editing the Extension's source code and observing the effects,
    3. comparing and copying files from the MyBB installation back into the source (e.g. a code repository)
  
    in which case static directory names would be preferred to preserve code history and allow quick comparison.
  - **System Complexity**

    A logical directory structure is preferred to simplify the implementation and related code (e.g. querying data using normalized paths), and to minimize cognitive load.

  Specifically, the following desired features can be identified, in order of importance:
  - **Overwrite Protection** (User Experience and Data Integrity — high importance)

    New versions of the Extension can be uploaded without overwriting past versions' data.
  - **Live Editing with Static Paths** (Developer Experience)

    The same file paths as the ones uploaded can be used to edit and preview changes.
  - **Importing with Static Paths** (Developer Experience)

    The Extension's files can be added to the installation without changing directory names to indicate specific versions beforehand.
  - **Exporting with Static Paths** (Developer Experience)

    The Extension's files can be downloaded and moved to the same directory structure without changing versioned directory names to static ones beforehand.
  - **Copy Synchronization** (Developer Experience)

    If copies of the Extension's files are made (e.g. to allow safe overwriting), changes are propagated to the copies properly.
  - **Normalized Paths** (System Complexity)

    The storage structure follows and illustrates the system's logical hierarchy.
  - **Recognition of Deleted Files** (User Experience and Data Integrity — low importance)

    Removed files, present in past versions, won't be incorrectly included again after uploading the new version.
  
  <br>

  Notably, the application is expected to be distributed with default Packages (such as the Core Theme), and the design of the System will affect the development workflow of the application itself.

  The following types of versioning methods were considered:
  
  - **Passive**, in which the directories are versioned before being used by the Extension System:
    - **Manual** — the simplest approach that puts the burden of naming Themelet directories according to their version on Extension developers
    - **On Detection** — an improved approach that allows developers to add (and distribute) Packages with static directory names, which are automatically renamed according to the Extension's version

  - **Active**, in which the target location with versioned Themelets is managed exclusively by the application, and Extension files are uploaded through the ACP or to a staging directory and processed by the application:
    - **Processing (Simple)** — Extension files are uploaded to the ACP or a temporary location and processed to rename directories according to the Extension's version
    - **Processing (Compatible)** — an improved approach that stores the latest Themelet version in a directory with a static name, allowing developers to access it for editing and exporting more easily

  - **Mixed**, in which administrators and developers can interact with the target location, but some intervention by the application is required:
    - **On Archiving** — the most recent Themelet is kept in a directory with a static name, and is renamed to `{version}/` through user action before a new version is uploaded
    - **Cold Duplicate** — Extension files are duplicated on detection to a `{version}/` directory; the static-named directory overrides content stored in `{version}/` directories as determined by the version specified in the manifest file; subsequent changes are propagated to the existing copy
    - **Hot Duplicate** — an improved approach, where Extension files from the static-named directory are synchronized to the `{version}/` copy, where data is accessed from; may be I/O-intensive, and would require user action to break synchronization before uploading new versions
  
  <br>
  
  **Table: Themelet Versioning Methods**
  
  Type | Versioning Method | Runtime Source Directory | Overwrite Protection | Recognition of Deleted Files | Importing with Static Paths | Live Editing with Static Paths | Copy Synchronization | Exporting with Static Paths 
  --|--|:--:|:--:|:--:|:--:|:--:|:--:|:--:
  n/a | _No Versioning_ | static path | ❌ No | ❌ No | ✅ Yes | ✅ Yes | n/a | ✅ Yes
  **Passive** | Manual | `{version}/` | ✅ Yes | ✅ Yes | ❌ No | ❌ No | n/a | ❌ No 
  **Passive** | On Detection | `{version}/` | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No | n/a | ❌ No 
  **Active** | Processing (Simple) | `{version}/` | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No | n/a | ❌ No 
  **Active** | Processing (Compatible) | static path (for latest version) or  `{version}/` (for older versions) | ✅ Yes | ✅ Yes | ✅ Yes | ⚠️ Yes; different location vs. import | n/a | ⚠️ Yes; different location vs. import
  **Mixed** | On Archiving | static path (for latest version) or  `{version}/` (for older versions) | ⚠️ Yes; action required in advance | ✅ Yes | ✅ Yes | ✅ Yes | n/a | ✅ Yes 
  **Mixed** | **Cold Duplicate** | static path (priority) or `{version}/` | ✅ Yes | ❌ No | ✅ Yes | ✅ Yes | ⚠️ Yes; action required| ✅ Yes 
  **Mixed** | Hot Duplicate | `{version}/` | ⚠️ Yes; action required in advance | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes 
  
  The _Cold Duplicate_ method was selected on the basis of desired features and feasibility.

  - #### _Cold Duplicate_ method
  
    Compared to _Passive_ and _Active_ methods, _Cold Duplicate_ allows to operate on standardized, static directory trees without experiencing unexpected changes, nor requiring additional processing, throughout the development cycle of a Package.

    Compared to other _Mixed_ methods, old Package versions residing under static paths are generally expected to be already archived when new versions are uploaded (v. _On Archiving_), and the copying always involves the application's logic (v. _Hot Duplicate_), offering better protection from accidental overwriting of archives by not requiring additional actions beforehand; instead, the need for intervention (manual or automated) is shifted to the moment after changes are made — at the cost of additional logic for keeping archives up to date with corresponding source files.

    Given that static paths — rather than versioned paths, or internal archives — are used for latest Package version references (v. _Passive_, _Active_, _Hot Duplicate_), basic features of the Theme System can use a simple _No Versioning_-like structure, and work independently of the archiving system, which itself can function as an optional background process.

    While this method doesn't handle deleted files implicitly, redundant Resources may be deduced with the implementation of Resource-level manifest files.

    <br>
  
    **Diagram: Themelet Version Addressing Priority**
  
    <img alt="themelet-version-addressing-priority" src="https://user-images.githubusercontent.com/8020837/155187159-125f6727-3651-44c0-9654-06c9430e95a1.png" width="50%">
  
    <br>
    <br>

    Directories with old Themelets (`{version}/`) may be stored under a separate path for practical reasons (see: [Directory Structure](#directory-structure)).
  
    Effectively, Resources supplied by a single Package may be stored in multiple locations:
    - source — directories that Extension developers are expected to interact with (existing in MyBB ≤ 1.8),
    - Themelet archives — directories with copies of source files managed by the archiving system,
    - web cache — directories with processed Resources saved for performance reasons, or available through HTTP (existing in MyBB ≤ 1.8).
  
    The directory structure will allow users to add old Themelets versions manually.
  
    ##### Copy Synchronization
    The _Cold Duplicate_ method relies on:
    - creating a copy (archive) for each new version of a Themelet, and
    - synchronizing existing copies (archives) with the source when changes occur.

    While Themelet operations (e.g. editing a Template) performed using the ACP involve the application's logic, which can reliably propagate them to version archives, the System is also expected to accept direct changes to source files of Packages. To support archiving in such scenarios, the implementation involves detecting changes that may have occurred, and addressing practical concerns related to copying files.
  
    - ###### Change Detection
      Unable to subscribe and immediately react to filesystem events, the archiving system polls Extension source directories according to a strategy that balances supporting common workflows — compatible with best-practice guidance and recommendations — and optimal performance.
      
      Given that third-party Packages are only expected to change with new versions (with users discouraged from modifying them directly), reliable support for comparison of changes in such parent Packages (which the user may be most unfamiliar with — compared to Packages developed locally — and benefit from such features most) can be achieved without additional user interaction by monitoring Extension metadata, expected to be loaded regularly by the Theme System, and thus without additional overhead.
      
      Other Packages — in which direct, individual Resource modifications may occur — require a more resource-intensive monitoring of numerous files. However, the importance of automated, non-interactive archiving of such Packages is relatively lower, as in each case one or more factors apply:
      - **First-party Authors**

        Authors of Packages being modified are more likely to recognize and understand changes between versions that may need to be propagated down the inheritance chain.

      - **User Experience**

        Increased technical experience and familiarity with the application suggests better receptiveness to communication and prompts (within the application and its documentation) related to interaction with the archiving system's controls.
      - **Development Environment**

        While propagation of changes to used Themes — and, effectively, comparison support — is especially important for Themelets of certain origins due to their association with back-end code (_Core Themes_, _Plugins_), the risk of unaddressed breaking changes and security problems applies minimally to development installations.

      It is, therefore, more acceptable to rely on interactive archiving, handled through controls in the ACP and configuration (e.g. action buttons, opt-in polling settings), or opportunistically — using data fetched by other components (e.g. file modification time that may be read to check status of other Resource caches).

      <br>
      
      **Table: Archiving System's Themelet Package Classification**
      Themelet Origin | Local Modifications | Expected Authors | Expected Environment | Expected User Experience | Resource File Polling
      -|-|-|-|-|-
      **Core Theme / Original Theme / Plugin** | **No (import only)** | Third-party | Development, Production | Low | No
      **Core Theme / Original Theme / Plugin** | **Yes** | First-party | Development | High | Yes
      **Versioned Board Theme** | **Yes** | First-party | Development, Production | Medium | Yes
      **Non-versioned Board Theme** | **Yes** | First-party | Production | Low | n/a (no archiving)

      <br>
      
      **Table: Themelet Characteristics by Origin**
      Themelet Origin | Update Prevalence | Back-end Code Dependency | Propagation Importance | Expected Authors | Expected Authoring Environment
      -|-|-|-|-|-
      Core Theme | High | Yes | High | Application developers | Development
      Plugin | Medium | Yes | High | Extension developers | Development
      Original Theme | Medium | No | Medium | Extension developers | Development
      Board Theme | Medium | No | Low | Webmasters | Production

      <br>

      **Table: Archive Health and Performance Impact by Synchronization Frequency**
      Frequency | Target | Detection Without User Interaction | Detection With User Interaction | Production Performance Impact
      -|-|-|-|-
      _Continuous (no architecture support)_ | All files | Best | n/a | n/a
      On application run | All files | Very Good | n/a | High
      Periodic polling | All files | Good | n/a | Medium
      Periodic polling (opt-in) | All files | n/a | Good | Low
      Manual triggers | All files | n/a | Very Good | Low
      On Resource query | Resource file | Good | n/a | Low/Medium *
      
      \* — depending on data used by the Theme System (or other application components)

    - ###### Partially-uploaded Packages
      The first step in the Package archiving logic involves identifying its version — provided in a Package manifest (metadata) file — to target the correct internal archive, to which the Themelet — provided in other Package files — is to be copied.
      
      Because the application — and the archiving logic — may be triggered during the process of copying or uploading new Package source files, only a subset of them may be detected until the process has finished. As files belonging to different versions of a Package must be placed in the same source directory by webmasters — overwriting the previous version — files associated with distinct versions may be mixed, resulting in a potential mismatch between the manifest file and Resource files.

      While partially-uploaded Packages with latest manifest files could be archived successfully by updating the archives continuously with added or overwritten source files, a scenario involving new Resources and old metadata may result in incorrectly writing new data the old version's archive.

      **Table: Basic Interpretation of Source Themelet Changes**

      x | Old Resource(s) | New/Modified Resource(s)
      --|--|--
      **Old Metadata** | No action | Copy to existing archive |
      **New Metadata** | Create new archive | Create new archive |

      Such mismatches may be avoided by:
      - linking the Package versions with Resources using checksums, and/or
      - delaying (debouncing) the archiving process until a predefined period of time has elapsed since the last modification of any Package file.

      <br>

      See: [Examples — Themelet Source-Archive Synchronization](#themelet-sourcearchive-synchronization)

    The ACP may be fitted with Extension authoring tools to supplement direct changes to source files, such as:
    - feedback relating to ongoing archiving process (e.g. time until archiving is triggered),
    - triggering the archiving process manually (shallow — by comparing file modification times, or deep — by comparing file content),
    - updating Resource checksums after manual changes.
    
    Due to potentially large numbers of Themelet files to be handled, the archiving system may benefit from extending the existing Task System to handle resource-intensive tasks separately (e.g. through automatic triggers, reducing its impact on regular HTTP requests).

  Although, as described above, the _Cold Duplicate_ method is preferred according to this spec, a mostly working version of the alternative _Processing (Compatible)_ method has been implemented in code given its perceived advantages.

  - #### _Processing (Compatible)_ method

    - Advantages and rationale:

      1. Minimises redundant storage: the current version of the theme is not archived until a new version is uploaded via the ACP.
      2. Ensures that the MyBB web app is immediately aware of all theme-related changes so that it can immediately perform such tasks as archiving themelets and rebuilding caches.
      3. Avoids the need for regular filesystem checks/monitoring.
      4. Avoids the possibility of archives being corrupted due to auto-archiving occurring in the middle of the uploading of a new version of a themelet, and thus avoids the need to handle this scenario via checksums and/or debouncing.
      5. Provides a smooth path towards our goal of in-app upgrades/installations of plugins and ultimately of MyBB core itself.

    - Summary of the current implementation:

      - Themes and plugins are installed and upgraded by an admin uploading each one as a ZIP archive via the relevant ACP page (_Themes_ and _Plugins_ respectively).
      - MyBB then unzips the archive to a temporary directory, performs integrity checks, and, if the checks pass, moves it to the `staging/themes/` or `staging/plugins` directory respectively.
      - If the theme/plugin is being upgraded (as opposed to being installed), then, next, MyBB archives the current version of its themelet - either `inc/themes/[codename]/current/` or `inc/plugins/[codename]/interface/current/` - to the `storage/themelets/[ext.][codename]/[version]/` directory (the "ext." prefix is only applicable for plugin themelets).
      - MyBB then "integrates" the staged theme/plugin by, firstly, moving the `devdist` version of its themelet to the relevant `current` directory (as above, and which on upgrade has just been archived), and then, secondly, for plugins, moving the remainder of its files to its live ("integration") directory, i.e., `inc/plugins/[codename]/`.
      - In the case of a theme, MyBB then auto-creates a (mutable) board theme from the uploaded original theme. In the case of a plugin, MyBB then installs and activates it.

    - Production and development modes:

      - As indicated above, there are two possible subdirectories of a themelet, whether as part of a plugin or of a theme proper: `current` and `devdist`. The `current` subdirectory is intended for production use, and is immutable except for board themes. The `devdist` subdirectory is intended for use (1) during development of the theme/plugin, and (2) for eventual distribution with the theme/plugin. It is mutable.
      - The mode can be toggled between production and development via a ubiquitous header of the ACP's _Templates & Style_ module, as well as via the corresponding ACP _General Configuration_ » _Development Mode (Themelets)_ setting. In production mode, MyBB uses the `current` directories for all theme-related functionality; conversely, in development mode, MyBB uses, where they exist, `devdist` directories for all theme-related functionality.
      - In development mode, developers can, via the ACP's _Templates & Style_ module, edit the properties and resources of a given themelet. These changes are written to the themelet's `devdist` directory.
      - When a developer is satisfied with the changes, a new version of the themelet can be exported via the ACP as an importable ZIP archive, based on the themelet's `devdist` directory (Note: exporting of plugins has not yet been implemented).
      - The developer can choose, on exporting the theme/plugin, for MyBB to automatically archive its `current` version and then recursively copy the contents of the `devdist` subdirectory to the now-empty `current` directory (Note: this feature has not yet been implemented).


- ### Resource Namespaces
  #### Avoiding Path Collisions
  Saving and accessing Resources from Themes and Plugins using a single namespace (i.e. without any logical separation) may lead to overlapping Resource paths.
  
  This results in Extensions "polluting" the general-use namespace, which may lead to accidental collisions of Resource paths (Theme–Plugin, Plugin–Plugin), e.g. when using common names in template groups. Even when Plugins follow best practices to contain own Resources within unique paths using codenames (e.g. `templates/{codename}/`), Theme–Plugin conflicts may occur when valid Extension codenames use common words that may be used in Themes' Resource paths.
  
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
  - undefined behavior (when a Plugin's Resources are accidentally overridden by another Plugin, given that Plugins don't have any hierarchy or explicit order).
  
  #### Separate Namespaces for Plugins
  To address this problem, Resources supplied by Plugin Themelets are placed in — and accessed from — their own namespace, which would result in distinct and unique paths managed by the Extension System.
  
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
  
  This design can, additionally, help prevent Plugins from overwriting Resources in chosen namespaces without doing so explicitly using Interface Macros (or other Theme modification functions), depending on chosen approach.
  
  Resource interactions (e.g. inheritance and overriding) will be handled separately inside each scope, taking into account the namespace, rather than just the Resource path (which may involve enforcing separate prefixes or directories for each scope). The namespace syntax used in the examples is [supported by Twig](https://twig.symfony.com/doc/3.x/api.html#twig-loader-filesystemloader).

  #### Source Priority in Extension Namespaces
  In addition to preventing accidental collisions, namespaces allow for better control of intentional overrides (compared to combining all Resources in a single namespace in undefined order).
  
  While Themes are expected to override Plugin Themelets (by assigning higher priority to Themes' contributions), namespaces allow for better control of external Plugins' contributions, such as:
  - disallowing Plugin–Plugin contributions completely (by not recognizing external Plugins' sources),
  - only allowing non-conflicting contributions (i.e. only new Resources) to another Plugin's namespace (by assigning higher priority to the target Plugin's original sources), or
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

  #### Generic Namespaces
  Resources typically supplied by Themes will be separated into namespaces according to interface type, such as:
  - the forum front-end,
  - ACP,
  - parser output,
  - email content, or
  - error messages.
  
  The separation may help with loading such modules individually (e.g. in the ACP, the forum front-end templates are not needed).

  <br>

  **Table: Themelet Namespace Interactions by Extension Type**
  
  Extension Type | Source in Generic Namespaces | Source in Own Extension Namespace | Source in Any Extension Namespace
  :--:|:--:|:--:|:--:
  **Core Theme** | ✅ Yes | no own namespace | ❌ No
  **Plugin** | ❌ No | ✅ Yes (highest/lowest priority, depending on chosen approach) | ❌ No / ➕ Add only / ✅ Yes (depending on chosen approach)
  **Original Theme** | ✅ Yes (overriding parents) | no own namespace | ✅ Yes (overriding parents)
  **Board Theme** | ✅ Yes (overriding parents) | no own namespace | ✅ Yes (overriding parents)
  
  <br>
  
  **Diagram: Example Registration of Interface Resources in Namespaces (Lowest to Highest Priority)**
  
  ![registration-of-interface-resources-in-namespaces](https://user-images.githubusercontent.com/8020837/154743960-8bbf9759-2b50-4a35-b18b-a918c834e791.png)
  
  <br>

- ### Directory Structure
  #### Hierarchy Order
  - Namespaces, Resource Types
    - A. Namespace-first (e.g. `frontend/templates/member/index.twig`)

         **✔ preferred** — containing Resources targeting the same interface type in the same repository; easier restructuring and reuse of Extension styling
    - B. Resource Type-first (e.g. `templates/frontend/member/profile.twig`)

  #### Hierarchy Levels
  The following storage choices were considered for each logical group in the hierarchy of Resources:
  
  1. Supplying Extension Type
     - A. Themelets stored in the Extension Directories (i.e. split between `inc/themes/` and `inc/plugins/`, according to type)

       **✔ preferred** — helping encapsulate Extension files in a single directory
     - B. Themelets stored in the same directory (e.g. `inc/themes/`, splitting the Extension's Themelet from other files)
  1. Extension's Target Interface Type/Namespace
     - A. Extension Directories spread across the filesystem according to type (e.g. `inc/themes/`, `admin/themes/`)
     - B. Extension Directories stored in subdirectories according to type (e.g. `inc/themes/frontend/`, `inc/themes/admin/`)
     - C. Extension Directories stored in the same directory for all types (e.g. `inc/themes/`) 

       **✔ preferred** — the same directory used for a single theme system handling both front-end and the ACP, and allowing Themes to target multiple types
  1. Extension Package
     - A. Extensions stored in directories referencing the Package Name (`<package-name>/`)
  1. Themelet Path
     - A. Themelets of all Extensions stored in standardized subdirectories (e.g. `interface/`) of Extension Directories
     - B. Themelets of Plugins stored in standardized subdirectories (e.g. `interface/`) of Extension Directories, and Themelets of Themes directly in Extension Directories

       **✔ preferred** — avoiding confusion with unrelated directories; simplifying directory structure for Themes
     - C. Themelets of all Extensions stored directly in Extension Directories
  1. Themelet Version
     - Latest Themelet Version
       - A. latest Themelets stored in subdirectories of Extension Themelet Paths (e.g. `current/`)
       - B. latest Themelets stored directly in Extension Themelet Paths

         **✔ preferred** — simplifying structure of directories exposed to developers and webmasters
     - Old Themelet Versions
       - A. old Themelets stored in an external structure (e.g. `cache/themelets/<package-name>/{version}/`)

         **✔ preferred** — removing necessity of loosening permissions of Extension directories
       - B. old Themelets stored in subdirectories of Extension Themelet Paths (e.g. `{version}/`)
  1. Target Namespace
     - Themes
       - A. all Themelets stored in directories named according to target namespace (`{namespace}/`) 

         **✔ required** — all Resources expected to belong to a namespace
     - Plugins
       - Own Namespace
         - A. directory with the Themelet targeting the Plugin's own namespace stored in subdirectory (equivalent to Themes)
         
           **✔ preferred** — own namespace directory on the same level as other targeted namespaces; structure similar to Themes; hinting to developers that Resources will be placed in a namespace
         - B. Themelet targeting the Plugin's own namespace stored directly in the Themelet directory
       - Other Namespaces
         - A. directories with Themelets targeting other Plugins' namespaces stored directly in the Themelet directory (e.g. `{namespace}/`)

           **✔ preferred** — directory on the same level as own namespace directory
         - B. directories with Themelets targeting other Plugins' namespaces stored in a standardized subdirectory (e.g. `extensions/{namespace}/`)
  1. Namespace Directory Name Prefix
     - A. `{namespace}/` (no prefix)

       **✔ preferred** — avoiding less-readable encoded characters in URLs in code repositories
     - B. `@{namespace}/`, compatible with Twig syntax

  1. Resource Type
     - A. Resources collected into a Type stored in directories identifying Type names (e.g. `templates/`)
  1. Resource Group
     - A. Resources collected into a Group stored in directories identifying Group names (e.g. `showthread/`)

- ### Resource Routing
  As Theme Packages are expected to contain all Resources in their own directories — themselves considered internal (not accessible through HTTP) — the application is required to process and propagate Resources from source directories to publicly-accessible locations.

  While making the initial copies (e.g. when activating a Theme Package) may be trivial for all types of Resources, further changes to source files may be missed when the Resources are queried without the involvement of the application's logic.

  During normal operation, the application exclusively handles Templates, and interprets declarations of individual processable Assets to append them to pages, but doesn't track references to other Resources (e.g. images).

  <br>

  **Table: Application Awareness of Immediate Resource Requests**
  Resource Type | Internal Cache | HTTP-exposed Cache | Application Logic
  -|-|-|-
  **Template** | Twig PHP Cache | ❌ No | ✅ Yes (server-side rendering)
  **Style / Script** | Optional | ✅ Yes | ✅ Yes (appending to returned documents)
  **Other** | Optional | ✅ Yes | ❌ No

  The missing coverage can be achieved through the consistent usage of the `asset_url()` Twig extension function in Templates (and equivalent resolution of Resource paths internally), which, in addition to returning the correct URLs depending on CDN settings during normal operation, may point selected requests to a PHP file that would return expected content from internal source files, and provide signals to the application to check source files for changes, when in development mode.

- ### JSON Data Files
  - #### `manifest.json`
    Defines metadata associated with an Extension. Expected to be compatible with the [composer.json schema](https://getcomposer.org/doc/04-schema.md#json-schema).

    Contains:
    - **codename** (`codename`)
    - **title** (`extra.title`), a human-friendly name displayed in the UI
    - **description** (`description`)
    - **version** (`version`), compatible with PHP's [`version_compare()`](https://www.php.net/manual/en/function.version-compare.php)
    - **authors** (`authors`), as defined by the Composer schema
    
    The `name` property is reserved for Package names used in the Composer/Packagist ecosystem.
    
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
  Themelet Resource type directories (e.g. `templates/`, `styles/`) may contain subdirectories separating Resources into groups (shown in an expandable/collapsible hierarchy in the ACP), which may be nested.

  The system may support translatable titles and descriptions for Resources and groups, provided in the JSON data files.

- ### Naming
  - **Theme Packages**
    
    Extension Package directories, in addition to uniquely identifying Packages, will be used to determine their origin:
    - system (using reserved static names, e.g. `core` or `core.default`)
    - instance-limited packages (using reserved, codename-incompatible format, referencing an internal ID, e.g. `theme.{id}` for Board Themes)
    - distributable packages (using the codename format enforced by the Extend platform: `[a-z_]+`)
  - **Namespaces**
    - Generic Namespaces: `{name}` (`[a-z_]+`)
    - Extension Namespaces: `ext.{codename}`

      Namespaces created for Plugins will use a prefix `ext.` to avoid collisions with Generic Namespaces.
  - **Namespace Directories**

    Resources supplied by a Plugin, intended for its own namespace, can be placed in an `ext/` directory, which will be assigned to the Plugin-specific namespace `ext.{codename}` automatically.
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

- #### Themelet Source–Archive Synchronization
  ```php
  function checkExtensionArchive(\MyBB\Extension $extension): void
  {
      if (
          !$extensionSystem->archiveExists(
              extension: $extension,
              version: $extension->getSourceVersion(),
          )
      ) {
          // version not present in archives
  
          $extensionSystem->createArchiveFromSource($extension);
      } else {
          // version present in archives
  
          $filesToUpdate = null;
  
          $outOfSyncFiles = $extensionSystem->getFilesOutOfSyncWithArchive($extension);
  
          if ($outOfSyncFiles !== []) {
              // changes made in source files of an archived version
  
              if ($extension->hasChecksumsFile()) {
                  // the package/system supports Resource checksums
  
                  /*
                   * if Resource checksums and manifest data are stored
                   * in different files, make sure they come from the same
                   * Package version
                   */
                  if ($extension->getSourceVersion() === $extension->getSourceChecksumsFileVersion()) {
  
                      // get files declared in the metadata file with matching checksums
                      $verifiedFiles = $extension->getVerifiedFiles();
  
                      $filesToUpdate = array_intersect($outOfSyncFiles, $verifiedFiles);
                  }
              } else {
                  $secondsSinceLastModified = TIME_NOW - $extension->getSourceLastModifiedDate();
  
                  // wait until uploads have likely finished
                  if ($secondsSinceLastModified >= ExtensionSystem::ARCHIVE_DEBOUNCE_TIME_SECONDS) {
                      $filesToUpdate = $outOfSyncFiles;
                  }
              }
      
              $extensionSystem->updateArchiveFromSource(
                  extension: $extension,
                  filesToUpdate: $filesToUpdate,
              );
          }
      }
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

- #### Theme System Directory Tree
  The following is an example of a partial directory tree related to the Theme System, featuring the internal Themelet cache and source files of:

  - Core Theme `core.default`, contributing to namespaces `frontend` and `parser`
  - Core Theme `core.acp`, contributing to namespace `acp`
  - Plugin `plugin_a`, contributing to its own namespace, and that of Plugin `plugin_b`
  - Theme `theme_a`, contributing to namespace `frontend`, and that of Plugin `plugin_b`

  ```
  .
  ├── data/
  │   └── themelets/
  │       ├── core.default/
  │       │   ├── 1.9.0/
  │       │   └── 1.9.1/
  │       ├── core.acp/
  │       │   ├── 1.9.0/
  │       │   └── 1.9.1/
  │       ├── plugin_a/
  │       │   └── 1.2.3/
  │       ├── plugin_b/
  │       │   └── 4.5.6/
  │       └── theme_a/
  │           └── 7.8.9/
  └── inc/
      ├── plugins/
      │   └── plugin_a/
      │       └── interface/
      │           ├── ext/
      │           │   ├── images/
      │           │   ├── scripts/
      │           │   ├── styles/
      │           │   ├── templates/
      │           │   └── resources.json
      │           └── ext.plugin_b/
      │               ├── templates/
      │               └── resources.json
      └── themes/
          ├── core.default/
          │   ├── frontend/
          │   │   ├── images/
          │   │   ├── scripts/
          │   │   ├── styles/
          │   │   ├── templates/
          |   |   |   └── path/
          |   |   |       └── template.twig
          │   │   └── resources.json
          │   ├── parser/
          │   │   ├── styles/
          │   │   ├── templates/
          │   │   └── resources.json
          │   ├── manifest.json
          │   └── properties.json
          ├── core.acp/
          │   ├── acp/
          │   │   ├── images/
          │   │   ├── scripts/
          │   │   ├── styles/
          │   │   ├── templates/
          │   │   └── resources.json
          │   └── manifest.json
          └── theme_a/
              ├── frontend/
              │   ├── styles/
              │   └── resources.json
              ├── ext.plugin_b/
              │   ├── styles/
              │   └── resources.json
              └── manifest.json
  ```

## General Roadmap
1. Fundamentals
   - move & add the default Theme's Resources according to the new filesystem structure
1. Theme System Basics
   - copying static files (CSS, images) to web-accessible directories
   - SCSS support
   - basic inheritance functionality
   - basic Theme Management functionality
   - Plugin Themelets
   - basic Theme Resource modification functions for Extensions
   - ACP Template & stylesheet editing
1. Theme System Improvements
   - Interface Macro API, Static Interface Macros
   - ACP UI for customizing values of CSS/SCSS variables
1. Advanced Theme System Features
   - Dynamic Interface Macros
   - Themelet archiving system
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
- **Web Root Directory**, `<web-root-directory>` — MyBB's main directory (referred to by PHP constant `MYBB_ROOT`)

*extensions*

- **Extension** — a component recognized by the application
  - - **Plugin** — a component intended to modify the application's behavior
    - **Theme** — a component defining the user interface
      - **Core Theme** — a Theme belonging to the Core, defining MyBB's default user interface; not editable during normal use
      - **Original Theme** — a Theme maintained by third-party; not editable during normal use
      - **Board Theme** — an editable Theme
  - **[Extension] Package** — a set of files associated with an Extension
  - **[Extension] Package Name**, `<package-name>` — a local or global (Extend platform) identifier of an Extension Package
  - **[Extension] Codename**, `<extension-codename>` — a unique identifier associated with an extension, limited to a-z letters and underscore `_`, registered on the Extend platform
  - **Extension Directory** — a directory containing files from an Extension Package, located directly under `inc/plugins/` or `inc/themes/` (see `<extension-directory-path>`)
  - **Extension Metadata** — information about the extension, such as the author and license, used in Extension Management
  - **Extension Manifest File** — a `manifest.json` file in JSON format, compatible with [Composer schema](https://getcomposer.org/doc/04-schema.md), containing Extension Metadata, located directly in an Extension Directory (see `<extension-manifest-file-path>`)
  - **Extension Property** — data associated with an Extension (e.g. color presets of a Theme)
  - **Extension Properties File** — a `properties.json` file in JSON format containing Extension Properties, located directly in an Extension Directory (see `<extension-properties-file-path>`)
- **Extension System** — the Theme System and the Plugin System
  - **Plugin System** — code responsible for execution of Plugins
    - **Plugins Directory** — the directory where source Plugin packages are stored (see `<plugins-directory-path>`)
  - **Theme System** — code responsible for execution and rendering of Themes
    - **Themes Directory** — the directory where source Theme packages are stored (see `<themes-directory-path>`)

- **Extension Management** — code and events related to configuring Extensions

*interface*

- **[Interface] Resource Namespace**, `<resource-namespace>` — the primary Resource organization unit
  - **Generic Namespace** — a Namespace containing Resources for a particular interface (e.g. front-end, ACP, email message, etc.)
  - **Extension Namespace** — a Plugin-specific Namespace
- **[Interface] Resource Type**, `<resource-type>` — a classification of Resources according to method of execution or interpretation
  - **Template** — code interpreted by a template engine
    - **PHP Template** — a MyBB 1.8-style Template interpreted using `eval()` statements
    - **Twig Template** — a MyBB 1.9-style Template interpreted by Twig
  - **Asset** — content and files intended to be exposed though HTTP
- **[Interface] Resource Group** — an organization unit for Resources of the same Type
- **[Interface] Resource** — an interface-related file intended to be processed or exposed through HTTP
  - **[Interface] Resource Property** — data associated with a Resource (e.g. pages to which a stylesheet is attached)

*theme system*

- **Themelet** — a collection of Interface Resources with their Properties
- **[Interface] Resource Properties File** — a `resources.json` file in JSON format containing Resource Properties (see `<resource-properties-file-path>`)
- **Themelet Directory** — a directory containing Themelet files (see `<themelet-directory-path>`)
- **Themelet Inheritance Chain** — an ordered list of Extensions, according to which a final Themelet is resolved, where priority is defined by the number of steps away from the end of the chain
- **Themelet Inheritance Base** — Themelet of the Core Theme and Plugin Themelets of active Plugins
- **Resolved Themelet** — Themelet collected or composed according to a defined Themelet Inheritance Chain
- **Compiled Themelet** — Resolved Themelet prepared for output or final interpretation (e.g. Twig templates converted to PHP code) and web-accessible static files

*macros*

- **[Interface] Macro** — a set of changes to a Themelet that can be stored, applied, and reversed
  - **Static [Interface] Macro** — an Interface Macro that operates on a Themelet and causes it to be stored in modified state
  - **Dynamic [Interface] Macro** — an Interface Macro that operates on a Resolved Themelet and causes a Compiled Themelet to be stored in modified state

*plugin interface*

- **Plugin Interface** — interface-related data supplied by a Plugin
  - **Plugin Themelet** — a Themelet supplied by a Plugin
  - **Plugin Interface Macro** — an Interface Macro supplied by a Plugin
- **[Plugin Interface] Macro Directory** — a directory containing Macro Files in subdirectories referencing targeted Extensions' Package names (see `<plugin-macro-directory-path>`)
- **[Plugin Interface] Macro File** — an `extend.php` file defining a Plugin Interface Macro, located in the Plugin Interface Macro Directory (see `<plugin-macro-file-path>`)
- **Plugin Interface Directory** — a directory containing Plugin Interface files (see `<plugin-interface-directory-path>`)


### [ABNF](https://datatracker.ietf.org/doc/html/rfc5234)
```abnf
; Interface Resources
resource-namespace           = 1*( a-z / "_" )           ; Generic Namespaces
                             / "ext." extension-codename ; Extension Namespaces
resource-namespace-directory = resource-namespace ; explicit Namespace
                             / "ext"              ; implicit Namespace
                             
resource-type             = "images"
                          / "scripts"
                          / "styles"
                          / "templates"
resource-path             = [resource-group "/"] resource-filename
resource-group            = 1*( a-z / "_" )
resource-filename         = 1*VCHAR "." 1*VCHAR ; name.extension

resource-path-in-themelet = resource-namespace-directory "/" resource-type "/" resource-path
absolute-resource-path    = <web-root-directory> "/" themelet-directory-path "/" resource-path-in-themelet

resource-properties-file-path = themelet-directory-path "/" resource-namespace-directory "/resources.json"

; Themelets
themelet-directory-path   = themelet-source-path
                          / themelet-archive-path

themelet-source-path      = theme-interface-directory-path  ; supplied a Theme
                          / plugin-interface-directory-path ; supplied by a Plugin

themelet-archive-path     = "data/themelets/" package-name "/" package-version

; Extensions
plugins-directory-path    = "inc/plugins"
themes-directory-path     = "inc/themes"

plugin-directory-path     = plugins-directory-path "/" plugin-package-name
theme-directory-path      = themes-directory-path "/" theme-package-name

plugin-interface-directory-path = plugin-directory-path "/interface"
theme-interface-directory-path  = theme-directory-path

extension-directory-path  = plugin-directory-path
                          / theme-directory-path

extension-manifest-file-path   = extension-directory-path "/manifest.json"
extension-properties-file-path = extension-directory-path "/properties.json"

package-name              = plugin-package-name
                          / theme-package-name
plugin-package-name       = extension-codename
theme-package-name        = "core." 1*( a-z / "_" ) ; Core Theme
                          / "theme." 1*DIGIT        ; Board Theme
                          / extension-codename      ; distributed Theme
extension-codename        = 1*( a-z / "_" )
package-version           = 1*( DIGIT / a-z / "." / "-" ) ; format supported by PHP's version-compare()

plugin-macro-directory-path = themelet-directory-path "/macros"
plugin-macro-file-path      = plugin-macro-directory-path "/" theme-package-name "/extend.php"
```

## References
- GitHub issue #3689 [1.9 Theme System](https://github.com/mybb/mybb/issues/3689)
- Community Forums thread [The 1.9 theme and template system](https://community.mybb.com/thread-232217.html)
- [Discord server](https://mybb.com/discord) discussions in `#1x-development`, `#staff-1x-organization` channels
