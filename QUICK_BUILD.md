# gtkpod Quick Build Reference

Quick reference for building gtkpod from source (for experienced users).

## One-Command Dependency Installation

```bash
sudo apt-get update && sudo apt-get install -y \
    autoconf automake libtool libtool-bin intltool pkg-config make gcc g++ flex gettext \
    libglib2.0-dev libgtk-3-dev libxml2-dev libgpod-dev libanjuta-dev libgdl-3-dev \
    libid3tag0-dev libcurl4-openssl-dev libflac-dev libvorbis-dev \
    libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev
```

## Critical Code Fixes Required

### Fix 1: libgtkpod/gtkpod_app_iface.h (lines 248-249)

```bash
sed -i 's/^GtkPodApp \*gtkpod_app;$/extern GtkPodApp *gtkpod_app;/' libgtkpod/gtkpod_app_iface.h
sed -i 's/^guint gtkpod_app_signals\[LAST_SIGNAL\];$/extern guint gtkpod_app_signals[LAST_SIGNAL];/' libgtkpod/gtkpod_app_iface.h
```

### Fix 2: libgtkpod/gtkpod_app_iface.c (after includes)

```bash
sed -i '/^#include "prefs.h"/a\
\
/* Global variables defined here */\
GtkPodApp *gtkpod_app = NULL;\
guint gtkpod_app_signals[LAST_SIGNAL] = {0};' libgtkpod/gtkpod_app_iface.c
```

### Fix 3: All Plugin Headers and Sources

```bash
cd plugins
for plugin in core_preferences details_editor info_display photo_editor repository_editor sjcd; do
  sed -i 's/^\([A-Za-z]*Plugin \*[a-z_]*_plugin\);$/extern \1;/' "${plugin}/plugin.h"

  case "$plugin" in
    core_preferences) var="CorePrefsPlugin *core_prefs_plugin = NULL;" ;;
    details_editor) var="DetailsEditorPlugin *details_editor_plugin = NULL;" ;;
    info_display) var="InfoDisplayPlugin *info_display_plugin = NULL;" ;;
    photo_editor) var="PhotoEditorPlugin *photo_editor_plugin = NULL;" ;;
    repository_editor) var="RepositoryEditorPlugin *repository_editor_plugin = NULL;" ;;
    sjcd) var="SJCDPlugin *sjcd_plugin = NULL;" ;;
  esac

  if ! grep -q "$var" "${plugin}/plugin.c"; then
    line_num=$(grep -n '#include "plugin.h"' "${plugin}/plugin.c" | cut -d: -f1 | head -1)
    [ -n "$line_num" ] && sed -i "${line_num}a\\
\\
/* Global plugin instance */\\
$var" "${plugin}/plugin.c"
  fi
done
cd ..
```

## Build Commands

```bash
./autogen.sh
./configure --disable-plugin-coverweb --disable-plugin-clarity --disable-plugin-sjcd --enable-deprecations
make -j$(nproc)
sudo make install
sudo ldconfig
```

## Post-Install

```bash
sudo glib-compile-schemas /usr/local/share/glib-2.0/schemas/
```

## Complete Build Script

Save as `build.sh`:

```bash
#!/bin/bash
set -e

echo "Installing dependencies..."
sudo apt-get update && sudo apt-get install -y \
    autoconf automake libtool libtool-bin intltool pkg-config make gcc g++ flex gettext \
    libglib2.0-dev libgtk-3-dev libxml2-dev libgpod-dev libanjuta-dev libgdl-3-dev \
    libid3tag0-dev libcurl4-openssl-dev libflac-dev libvorbis-dev \
    libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev

echo "Applying code fixes..."

# Fix libgtkpod header
sed -i 's/^GtkPodApp \*gtkpod_app;$/extern GtkPodApp *gtkpod_app;/' libgtkpod/gtkpod_app_iface.h
sed -i 's/^guint gtkpod_app_signals\[LAST_SIGNAL\];$/extern guint gtkpod_app_signals[LAST_SIGNAL];/' libgtkpod/gtkpod_app_iface.h

# Fix libgtkpod source
sed -i '/^#include "prefs.h"/a\
\
/* Global variables defined here */\
GtkPodApp *gtkpod_app = NULL;\
guint gtkpod_app_signals[LAST_SIGNAL] = {0};' libgtkpod/gtkpod_app_iface.c

# Fix all plugins
cd plugins
for plugin in core_preferences details_editor info_display photo_editor repository_editor sjcd; do
  sed -i 's/^\([A-Za-z]*Plugin \*[a-z_]*_plugin\);$/extern \1;/' "${plugin}/plugin.h"

  case "$plugin" in
    core_preferences) var="CorePrefsPlugin *core_prefs_plugin = NULL;" ;;
    details_editor) var="DetailsEditorPlugin *details_editor_plugin = NULL;" ;;
    info_display) var="InfoDisplayPlugin *info_display_plugin = NULL;" ;;
    photo_editor) var="PhotoEditorPlugin *photo_editor_plugin = NULL;" ;;
    repository_editor) var="RepositoryEditorPlugin *repository_editor_plugin = NULL;" ;;
    sjcd) var="SJCDPlugin *sjcd_plugin = NULL;" ;;
  esac

  if ! grep -q "$var" "${plugin}/plugin.c"; then
    line_num=$(grep -n '#include "plugin.h"' "${plugin}/plugin.c" | cut -d: -f1 | head -1)
    [ -n "$line_num" ] && sed -i "${line_num}a\\
\\
/* Global plugin instance */\\
$var" "${plugin}/plugin.c"
  fi
  echo "Fixed $plugin"
done
cd ..

echo "Generating build system..."
./autogen.sh

echo "Configuring..."
./configure \
    --disable-plugin-coverweb \
    --disable-plugin-clarity \
    --disable-plugin-sjcd \
    --enable-deprecations

echo "Building..."
make -j$(nproc)

echo "Installing..."
sudo make install
sudo ldconfig
sudo glib-compile-schemas /usr/local/share/glib-2.0/schemas/

echo "Build complete! Run with: gtkpod"
```

Make executable and run:

```bash
chmod +x build.sh
./build.sh
```

## Common Issues

| Issue | Solution |
|-------|----------|
| Multiple definition errors | Apply all code fixes above |
| libgtkpod.so.1 not found | `sudo ldconfig` |
| Schema not installed | `sudo glib-compile-schemas /usr/local/share/glib-2.0/schemas/` |
| Deprecated warnings | Normal, use `--enable-deprecations` |
