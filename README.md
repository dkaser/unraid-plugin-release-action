# unraid-plugin-release-action

This GitHub Action automates the process of building, packaging, and releasing Unraid plugins from your repository. It supports custom build steps, changelog generation, and flexible configuration for a variety of plugin types.

## Action Options

The action supports several options via workflow inputs:

- `github_token` (required): GitHub token for authentication.
- `plg_branch`: Branch to use for the templated plugin file (default: `main`).
- `plugin_version`: Version of the plugin (default: release name).
- `composer_dir`: If set, runs `composer install` in the specified directory.
- `go_dir`: If set, sets up Go using the `go.mod` and `go.sum` in the specified directory.
- `build_script`: Bash script to run before building the Slackware package. Use this to prepare files in `src/`.
- `changelog_releases`: Number of releases (including the current one) to include in the plugin changelog.

## Action Files

### plugin/plugin.json

This file defines metadata and configuration for your plugin.  
See the sample [plugin.json](plugin-example/plugin.json) for a complete example.

- `name`: (Required) Used for the plugin file name.
- `package_name`: (Required) Used for the Slackware package.
- Additional fields can be added and referenced in your template.

### plugin/plugin.j2

This is a Jinja2 template used to generate the Unraid `.plg` file.  
See the sample [plugin.j2](plugin-example/plugin.j2) for a complete example.

You can reference fields from `plugin.json` and environment variables set by the action.

## Creating the GitHub Workflow

Add a workflow file (e.g., `.github/workflows/release.yml`) with the following content:

```yaml
name: Release Unraid Plugin

on:
  release:
    types: [published]

jobs:
  build-and-release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Release Unraid Plugin
        uses: dkaser/unraid-plugin-release-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          # Optional:
          # plg_branch: main
          # composer_dir: plugin-php
          # go_dir: plugin-go
          # build_script: ./build.sh
          # changelog_releases: 5
```
## Releasing the Plugin

To release your plugin, create a new GitHub release with the following guidelines:

### Release Configuration

- **Tag**: Use the plugin version number
- **Name**: Use the plugin version number  
- **Description**: Describe the plugin changes (this content will be included in the plugin's changelog)

### Version Numbering

**Important:** Slackware packages do not work well with letters in version numbers.

❌ **Avoid:**
- `2025.01.01a`
- `2025.01.01b`

✅ **Use instead:**
- `2025.01.01.1`
- `2025.01.01.2`

If you need to release multiple plugin updates on the same day, append an additional number rather than using letters.

## Repository Structure

Your repository should include the following structure:

```
plugin/
  plugin.json   # Plugin metadata and configuration
  plugin.j2     # Jinja2 template for the .plg file
src/
  install/
    slack-desc  # Description of the slackware package
    doinst.sh   # Script run after package files are extracted (optional)
  usr/
    local/
      emhttp/
        plugins/
          my.plugin.name/  # Your plugin files go here
            README.md
            MyPlugin.page
            include/
            ...
  ...           # Other files needed by the plugin
```

### Understanding the `src/` Directory Structure

The `src/` directory contains your plugin files organized exactly as they should appear on the Unraid system after installation:

- **The `plugin/` directory** contains the configuration and template files required for the action.
- **The `src/` directory** should be structured as a Slackware package, with folders mirroring the final installation paths on Unraid.

**Important:** Any folders included in `src/` (other than the special `install/` folder) are extracted to the root of the Unraid installation when the package is installed. This means:

- Files in `src/usr/local/emhttp/plugins/my.plugin.name/` will be installed to `/usr/local/emhttp/plugins/my.plugin.name/` on the Unraid system
- Files in `src/etc/rc.d/` would be installed to `/etc/rc.d/` on the Unraid system
- And so on...

## Common Plugin Development Patterns

### Minimal Plugin Structure

For a basic plugin that adds a page to the Unraid WebGUI:

```
src/
  usr/
    local/
      emhttp/
        plugins/
          my.plugin.name/
            README.md           # Plugin documentation
            MyPlugin.page       # Main plugin page (appears in WebGUI)
  install/
    slack-desc               # Package description
```

### Plugin with Background Service

For a plugin that runs a background service or daemon:

```
src/
  usr/
    local/
      emhttp/
        plugins/
          my.plugin.name/
            MyPlugin.page       # Main plugin page
            scripts/
              start.sh          # Service start script
              stop.sh           # Service stop script
  etc/
    rc.d/
      rc.my.plugin.name         # System service script
  install/
    slack-desc
    doinst.sh                   # Starts service
```

## Slackware Packaging Files

The `src/` directory should be organized as a Slackware package. Slackware is the package management system used by Unraid, and understanding its basic concepts will help you structure your plugin correctly.

### What is Slackware Packaging?

Slackware packages are simply compressed archives (`.txz` files) that contain:

1. Your plugin files organized in the exact directory structure they should have on the target system
2. Optional installation script
3. Package metadata

When a Slackware package is installed, the files are extracted to their corresponding locations on the filesystem. For example, a file at `usr/local/emhttp/plugins/myplugin/default.php` in the package will be placed at `/usr/local/emhttp/plugins/myplugin/default.php` on the Unraid system.

### Required and Optional Files

Two important files for Slackware packaging are:

- **slack-desc** (required):  
  A text file located at `src/install/slack-desc` that provides a description of your package.  
  This file has a specific format with exactly 11 lines, where the first line contains a "handy ruler" for alignment.

  **Format explanation:**

  - Line 1: A ruler showing the 70-character width limit
  - Lines 2-11: Package description lines, each starting with `package-name:`
  - Each description line can be up to 70 characters total
  - Empty lines should still include the `package-name:` prefix

  See the [Slackware documentation](https://docs.slackware.com/slackware:package_management) for format details.  
  Example:

  ```
               |-----handy-ruler------------------------------------------------------|
  unraid-plugin: unraid-plugin (Short Description)
  unraid-plugin:
  unraid-plugin: This is a longer description of what your plugin does. You can
  unraid-plugin: spread the description across multiple lines, but keep each line
  unraid-plugin: under 70 characters including the prefix.
  unraid-plugin:
  unraid-plugin: Homepage: https://github.com/example/unraid-plugin
  unraid-plugin:
  unraid-plugin:
  unraid-plugin:
  unraid-plugin:
  ```

  **Important:** The prefix `unraid-plugin:` MUST exactly match the `package_name` field in your `plugin/plugin.json` file.

- **doinst.sh** (optional):  
  A shell script located at `src/install/doinst.sh` that is executed **after** the package files are extracted to the filesystem.  
  Use this for any post-installation setup your plugin requires, such as:

  - Creating additional directories
  - Setting file permissions
  - Starting services
  - Migrating configuration from previous versions

  Example:

  ```bash
  #!/bin/bash
  # Create plugin data directory
  mkdir -p /boot/config/plugins/my.plugin.name

  # Set proper permissions for scripts
  chmod +x /usr/local/emhttp/plugins/my.plugin.name/scripts/*.sh

  # Start plugin service if not running
  if ! pgrep -f "my.plugin.daemon" > /dev/null; then
    /usr/local/emhttp/plugins/my.plugin.name/scripts/start.sh
  fi
  ```

### File Permissions

When organizing files in the `src/` directory:

- Make sure any executable files (shell scripts, binaries) are marked as executable in your git repository using `chmod +x filename`
- The packaging process preserves file permissions from your repository
- If you need to set permissions during installation, use the `doinst.sh` script

## Notes

### For Plugin Authors New to Slackware Packaging:

- **File Placement:** Remember that the `src/` directory structure mirrors exactly where files will be placed on the Unraid system. If you want a file at `/usr/local/emhttp/plugins/myplugin/test.php`, place it at `src/usr/local/emhttp/plugins/myplugin/test.php` in your repository.

- **The `install/` Directory is Special:** The `src/install/` directory is the only exception to the direct file placement rule. Files in this directory are not copied to the filesystem; instead, they contain installation metadata and scripts that run during package installation.

- **Plugin Naming:** Use consistent naming throughout:

  - Your `package_name` in `plugin/plugin.json`
  - The prefix in your `src/install/slack-desc` file
  - Your plugin directory name under `src/usr/local/emhttp/plugins/`

- **Executable Files:** Any executable files should be marked as executable (`chmod +x`) in your repository. This ensures they are correctly marked in the Slackware package.

### Testing:

- You can test your plugin structure by manually copying the contents of `src/` (except `install/`) to the root of a test Unraid system to verify file placement.

### Build Process:

- The action will generate a `.plg` file using your template and metadata, and upload the package to the GitHub release. The templated plugin file is committed to the branch specified by `plg_branch`.
- The `src/` directory should follow Slackware packaging conventions, including any required install scripts and folder structure.
- A `.txz` package file will be created containing all files from your `src/` directory, preserving the directory structure and file permissions.

## License

See [LICENSE](LICENSE) for details.
