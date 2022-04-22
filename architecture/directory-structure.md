# MyBB Directory Structure
This document specifies the target directory structure of the MyBB application.

## Objectives
- introduce a strict separation of files by type, i.a. data, source (application and Extensions), and other (e.g. temporary cache)
- separate HTTP-accessible files from internal ones (improving security and avoiding `index.html` and server configuration files)
- separate the ACP interface (allowing additional access control, and taking advantage of browsers' same-origin policy to create a boundary between the two interfaces)

## Structure
### Partial Example
<!--
bin/
    cli
cache/
    settings.php
data/
    backups/
    themelets/
    uploads/
    config.php
ext/
    languages/
    plugins/
    themes/
public/
    admin/
        static/
        .htaccess
        index.php
    install/
        static/
        .htaccess
        index.php
    assets/
    .htaccess
    index.php
src/
    global.php
vendor/
-->
```
.
├── bin/
│   └── cli
├── cache/
│   └── settings.php
├── data/
│   ├── backups/
│   ├── themelets/
│   ├── uploads/
│   └── config.php
├── ext/
│   ├── languages/
│   ├── plugins/
│   └── themes/
├── public/
│   ├── admin/
│   │   ├── assets/
│   │   ├── .htaccess
│   │   └── index.php
│   ├── install/
│   │   ├── assets/
│   │   ├── .htaccess
│   │   └── index.php
│   ├── assets/
│   ├── .htaccess
│   └── index.php
├── src/
│   └── global.php
└── vendor/
```

### Main Directories
- ### `bin/`
   Scripts and executables, including MyBB's main command-line interface `cli`.

  **Naming:** in accordance with common practice.

- ### `cache/`

  Temporary files that may be deleted without loss of data or functionality.

  **Naming:** best simple description.

- ### `data/`

  Permanent storage for internal data.

  **Naming:** best simple description.

- ### `ext/`

  Plugin, Theme, and Language packages.

  **Naming:** in accordance with Resource namespace prefix `ext`.

- ### `public/`

  HTTP-accessible files (routing PHP files and public cache).

  **Naming:** in accordance with common practice.

- ### `src/`

  The application's own source code.

  **Naming:** in accordance with common practice.

- ### `vendor/`

  Third-party dependencies.

  **Naming:** in accordance with Composer defaults.

## Implementation Suggestions
- support path configuration in `config.php`, `index.php`, other file, or environment variables
- support moving `public/` content one level up
- support moving internal directories to a single directory for easier organization and access control
- include detection and advice in the installer for proper configuration
- include a helper `index.html` file to route/instruct users if the application was uploaded incorrectly
