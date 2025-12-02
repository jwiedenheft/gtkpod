# gtkpod Compilation and Installation Guide

This guide walks you through compiling and installing gtkpod from source on a Debian/Ubuntu-based system.

## Prerequisites

### System Requirements
- Debian 12 (Bookworm) or Ubuntu 22.04+ (or similar distributions)
- At least 500 MB of free disk space for build dependencies
- Internet connection for downloading packages

### Required Build Tools

First, update your package manager and install the essential build tools:

```bash
sudo apt-get update
sudo apt-get install -y \
    autoconf \
    automake \
    libtool \
    libtool-bin \
    intltool \
    pkg-config \
    make \
    gcc \
    g++ \
    flex \
    gettext
```

### Required Libraries

Install the core dependencies required by gtkpod:

```bash
sudo apt-get install -y \
    libglib2.0-dev \
    libgtk-3-dev \
    libxml2-dev \
    libgpod-dev \
    libanjuta-dev \
    libgdl-3-dev \
    libid3tag0-dev
```

### Optional Dependencies (Recommended)

These enable additional features like audio conversion and media playback:

```bash
sudo apt-get install -y \
    libcurl4-openssl-dev \
    libflac-dev \
    libvorbis-dev \
    libgstreamer1.0-dev \
    libgstreamer-plugins-base1.0-dev
```

## Obtaining the Source Code

Clone the gtkpod repository or extract the source tarball:

```bash
# If using git:
git clone https://github.com/YOUR_REPO/gtkpod.git
cd gtkpod

# Or if you have a tarball:
tar -xzf gtkpod-X.X.X.tar.gz
cd gtkpod-X.X.X
```

## Build Process

### Step 1: Generate Build System

Run the autogen script to generate the configure script:

```bash
./autogen.sh
```

**Note:** You may see warnings about deprecated macros - these are normal and can be ignored.

### Step 2: Fix Known Compilation Issues

Before configuring, we need to fix some header files that cause "multiple definition" errors with modern compilers.

#### Fix 1: Global Variables in libgtkpod

Edit `libgtkpod/gtkpod_app_iface.h` and change lines 248-249 from:

```c
GtkPodApp *gtkpod_app;
guint gtkpod_app_signals[LAST_SIGNAL];
```

to:

```c
extern GtkPodApp *gtkpod_app;
extern guint gtkpod_app_signals[LAST_SIGNAL];
```

Then edit `libgtkpod/gtkpod_app_iface.c` and add these definitions after the `#include` statements (around line 39):

```c
/* Global variables defined here */
GtkPodApp *gtkpod_app = NULL;
guint gtkpod_app_signals[LAST_SIGNAL] = {0};
```

#### Fix 2: Plugin Global Variables

For each plugin, apply similar fixes:

**Core Preferences Plugin:**
```bash
# Edit plugins/core_preferences/plugin.h
# Change: CorePrefsPlugin *core_prefs_plugin;
# To: extern CorePrefsPlugin *core_prefs_plugin;

# Edit plugins/core_preferences/plugin.c
# Add after #include "plugin.h":
# /* Global plugin instance */
# CorePrefsPlugin *core_prefs_plugin = NULL;
```

**Details Editor Plugin:**
```bash
# Edit plugins/details_editor/plugin.h
# Change: DetailsEditorPlugin *details_editor_plugin;
# To: extern DetailsEditorPlugin *details_editor_plugin;

# Edit plugins/details_editor/plugin.c
# Add after #include "plugin.h":
# /* Global plugin instance */
# DetailsEditorPlugin *details_editor_plugin = NULL;
```

**Info Display Plugin:**
```bash
# Edit plugins/info_display/plugin.h
# Change: InfoDisplayPlugin *info_display_plugin;
# To: extern InfoDisplayPlugin *info_display_plugin;

# Edit plugins/info_display/plugin.c
# Add after #include "plugin.h":
# /* Global plugin instance */
# InfoDisplayPlugin *info_display_plugin = NULL;
```

**Photo Editor Plugin:**
```bash
# Edit plugins/photo_editor/plugin.h
# Change: PhotoEditorPlugin *photo_editor_plugin;
# To: extern PhotoEditorPlugin *photo_editor_plugin;

# Edit plugins/photo_editor/plugin.c
# Add after #include "plugin.h":
# /* Global plugin instance */
# PhotoEditorPlugin *photo_editor_plugin = NULL;
```

**Repository Editor Plugin:**
```bash
# Edit plugins/repository_editor/plugin.h
# Change: RepositoryEditorPlugin *repository_editor_plugin;
# To: extern RepositoryEditorPlugin *repository_editor_plugin;

# Edit plugins/repository_editor/plugin.c
# Add after #include "plugin.h":
# /* Global plugin instance */
# RepositoryEditorPlugin *repository_editor_plugin = NULL;
```

**SJCD Plugin:**
```bash
# Edit plugins/sjcd/plugin.h
# Change: SJCDPlugin *sjcd_plugin;
# To: extern SJCDPlugin *sjcd_plugin;

# Edit plugins/sjcd/plugin.c
# Add after #include "plugin.h":
# /* Global plugin instance */
# SJCDPlugin *sjcd_plugin = NULL;
```

**Quick Fix Script:**

Alternatively, run this script to fix all plugins automatically:

```bash
cd plugins

for plugin in core_preferences details_editor info_display photo_editor repository_editor sjcd; do
  # Fix header
  sed -i 's/^\([A-Za-z]*Plugin \*[a-z_]*_plugin\);$/extern \1;/' "${plugin}/plugin.h"

  # Determine variable name and type
  case "$plugin" in
    core_preferences) var="CorePrefsPlugin *core_prefs_plugin = NULL;" ;;
    details_editor) var="DetailsEditorPlugin *details_editor_plugin = NULL;" ;;
    info_display) var="InfoDisplayPlugin *info_display_plugin = NULL;" ;;
    photo_editor) var="PhotoEditorPlugin *photo_editor_plugin = NULL;" ;;
    repository_editor) var="RepositoryEditorPlugin *repository_editor_plugin = NULL;" ;;
    sjcd) var="SJCDPlugin *sjcd_plugin = NULL;" ;;
  esac

  # Fix source file if not already fixed
  if ! grep -q "$var" "${plugin}/plugin.c"; then
    line_num=$(grep -n '#include "plugin.h"' "${plugin}/plugin.c" | cut -d: -f1 | head -1)
    if [ -n "$line_num" ]; then
      sed -i "${line_num}a\\
\\
/* Global plugin instance */\\
$var" "${plugin}/plugin.c"
      echo "Fixed $plugin"
    fi
  fi
done

cd ..
```

### Step 3: Configure the Build

Configure the build with optional plugins disabled (some require additional dependencies):

```bash
./configure \
    --disable-plugin-coverweb \
    --disable-plugin-clarity \
    --disable-plugin-sjcd \
    --enable-deprecations
```

**Configuration Options:**

- `--disable-plugin-coverweb` - Disables web-based cover art browser (requires WebKit)
- `--disable-plugin-clarity` - Disables OpenGL cover art display (requires Clutter)
- `--disable-plugin-sjcd` - Disables Sound Juicer CD ripping (requires Brasero libraries)
- `--enable-deprecations` - Allows deprecated GTK functions (needed for GTK 3.10+)

To see all available options:

```bash
./configure --help
```

### Step 4: Compile

Build gtkpod using all available CPU cores:

```bash
make -j$(nproc)
```

**Expected Warnings:**

You may see warnings about deprecated functions during compilation. These are normal and don't prevent successful compilation:

- `'GTimeVal' is deprecated`
- `'GtkStock' is deprecated`
- `'gtk_image_menu_item_*' is deprecated`
- `'gtk_menu_popup' is deprecated`

These warnings are due to the codebase using older GTK APIs but don't affect functionality.

### Step 5: Install

Install gtkpod system-wide:

```bash
sudo make install
sudo ldconfig
```

**Installation Locations:**

- Binary: `/usr/local/bin/gtkpod`
- Libraries: `/usr/local/lib/libgtkpod.so.*`
- Plugins: `/usr/local/lib/gtkpod/`
- Data files: `/usr/local/share/gtkpod/`
- Documentation: `/usr/local/share/gtkpod/doc/`
- Man page: `/usr/local/share/man/man1/gtkpod.1`

### Step 6: Install GSettings Schema

The GSettings schema is automatically installed, but you may need to compile it:

```bash
sudo glib-compile-schemas /usr/local/share/glib-2.0/schemas/
```

## Verification

Check that gtkpod is installed correctly:

```bash
which gtkpod
# Should output: /usr/local/bin/gtkpod

gtkpod --help
# Should display usage information
```

## Running gtkpod

### With GUI (requires X11/Wayland):

```bash
gtkpod
```

### With an iPod mounted at a specific location:

```bash
gtkpod --mountpoint=/media/ipod
```

### Command-line options:

```bash
gtkpod --help               # Show help
gtkpod --hash FILE          # Print gtkpod hash for file
gtkpod --playcount FILE     # Increment playcount for file
```

## Troubleshooting

### Issue: "libgtkpod.so.1: cannot open shared object file"

**Solution:** Run `sudo ldconfig` or add `/usr/local/lib` to your library path:

```bash
echo "/usr/local/lib" | sudo tee /etc/ld.so.conf.d/local.conf
sudo ldconfig
```

### Issue: "Settings schema 'org.gtkpod' is not installed"

**Solution:** Compile the GSettings schemas:

```bash
sudo glib-compile-schemas /usr/local/share/glib-2.0/schemas/
```

### Issue: Compilation fails with "multiple definition" errors

**Solution:** Make sure you applied all the header file fixes in Step 2.

### Issue: "gtk_menu_popup is deprecated" warnings

**Solution:** This is expected. The `--enable-deprecations` flag allows these warnings. They don't affect functionality.

### Issue: Warnings about locale, accessibility bus, or dconf

**Solution:** These are normal in headless/server environments. They don't prevent the program from running.

## Uninstalling

To remove gtkpod from your system:

```bash
cd /path/to/gtkpod/source
sudo make uninstall
sudo ldconfig
```

## Building Without Installation (Development)

To test gtkpod without installing:

```bash
# Set library path
export LD_LIBRARY_PATH=/path/to/gtkpod/libgtkpod/.libs:$LD_LIBRARY_PATH

# Run from build directory
src/.libs/gtkpod
```

## Feature Matrix

Based on the build configuration, the following features are available:

| Feature | Status | Notes |
|---------|--------|-------|
| Media Player | ✅ Enabled | Requires GStreamer |
| MP4 File Type | ✅ Enabled | |
| FLAC File Type | ✅ Enabled | Requires libflac |
| Ogg File Type | ✅ Enabled | Requires libvorbis |
| Cover Download | ✅ Enabled | Requires libcurl |
| M4A Conversion | ❌ Disabled | Requires faad |
| CoverWeb Browser | ❌ Disabled | Requires WebKit |
| Clarity Display | ❌ Disabled | Requires Clutter |
| Sound Juicer CD Ripping | ❌ Disabled | Requires Brasero |

## Additional Notes

### About Deprecated GTK Functions

gtkpod was originally written for GTK 2.x and has been partially ported to GTK 3.x. Some deprecated GTK stock items and functions are still used, which causes warnings but doesn't affect functionality. The `--enable-deprecations` flag is necessary for compilation with modern GTK versions.

### Headless Environment

gtkpod is primarily a GUI application. In headless environments (no display server), you'll see warnings but can still use command-line features. For full GUI functionality, ensure you have:

- A display server (X11 or Wayland)
- Or use X11 forwarding: `ssh -X user@host`
- Or use VNC/RDP

### Contributing Fixes

If you improve upon these fixes or update the code for newer GTK versions, consider contributing back to the gtkpod project.

## Support

For issues not covered in this guide:

- Check the TROUBLESHOOTING file in the source directory
- Read the README file for general information
- Visit the gtkpod documentation: `/usr/local/share/gtkpod/doc/`
- Man page: `man gtkpod`

## License

gtkpod is licensed under the GNU General Public License v2 (GPLv2). See the COPYING file in the source directory for details.
