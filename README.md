# nomacs — Image Lounge 🍸 (personal macOS fork)

> This is a **personal fork** of [nomacs/nomacs](https://github.com/nomacs/nomacs).
> The README below is trimmed down to a single thing: a **repeatable macOS (Apple
> Silicon) build-and-install runbook** that actually produces a working, self-contained
> `nomacs.app` in `/Applications`. It documents every fix that the upstream macOS
> instructions are missing on a current Apple Silicon Mac (macOS 26 / Tahoe, Homebrew Qt6).
>
> nomacs is a free, open-source image viewer (RAW, PSD, and most common formats),
> licensed under the **GNU General Public License v3** — see [`LICENSE`](LICENSE).
> Upstream project: [nomacs.org](https://nomacs.org) · [GitHub](https://github.com/nomacs) · [Matrix](https://matrix.to/#/#nomacs:matrix.org).

---

## TL;DR for Claude

If you are Claude Code running on a fresh Apple Silicon Mac and asked to "install nomacs",
follow the **numbered steps** below top-to-bottom. They are ordered and each explains *why*
so you can adapt if a path/version differs. The non-obvious failure modes are called out in
**⚠️ Gotcha** blocks — those are the parts the upstream README does not cover. The end state is:

- `/Applications/nomacs.app` — self-contained (embeds Qt + all deps, loads **zero** Homebrew
  libs at runtime), runs from Finder with no quarantine prompt.
- `build/nomacs.dmg` — a drag-install disk image of the same app.

Verified working: **macOS 26 (Darwin 25.x), Apple Silicon (arm64), Homebrew Qt 6.11, nomacs 3.23**.

---

## 0. Prerequisites

- Apple Silicon Mac (these paths assume Homebrew at `/opt/homebrew`; on Intel it's `/usr/local`).
- Xcode **Command Line Tools** (full Xcode is *not* required): `xcode-select --install`
- [Homebrew](https://brew.sh).

## 1. Get the source (with submodules)

```bash
git clone https://github.com/ivanearisty/nomacs.git
cd nomacs
git submodule update --init --recursive   # pulls plugins + 3rd-party libs
```

## 2. Install dependencies

```bash
brew install qt6 exiv2 opencv libraw quazip cmake pkg-config ninja
```

> `ninja` is not in the upstream list but is **required** later by `make kimageformats`.

## 3. ⚠️ Gotcha — fix the Homebrew QuaZip `libbz2.tbd` link bug

The Homebrew QuaZip bottle hardcodes the **build server's Xcode SDK path** for `libbz2.tbd`
in its exported CMake target. On a Command-Line-Tools-only Mac that path does not exist, so
`make` dies with: `No rule to make target '.../MacOSX*.sdk/usr/lib/libbz2.tbd'`.
Replace the absolute path with a plain `bz2` link flag:

```bash
QZ=$(find /opt/homebrew/opt/quazip/lib/cmake -name 'QuaZip-Qt6_SharedTargets.cmake')
cp -n "$QZ" "$QZ.bak"
sed -i '' -E 's#/[^";]*MacOSX[^";]*/libbz2\.tbd#bz2#' "$QZ"
grep -n bz2 "$QZ"   # INTERFACE_LINK_LIBRARIES should now end in ";bz2"
```

*(Reverts on `brew upgrade quazip`; just re-run this if you upgrade.)*

## 4. Configure with CMake

⚠️ You **must** add OpenCV to `CMAKE_PREFIX_PATH` or configure fails with
"Could not find OpenCVConfig.cmake".

```bash
mkdir -p build && cd build
CMAKE_PREFIX_PATH="/opt/homebrew/opt/qt6/lib/cmake:/opt/homebrew/opt/opencv/lib/cmake/opencv4" \
  cmake -D ENABLE_QUAZIP=ON ../ImageLounge
```

You should see a summary with `OPENCV … YES`, `LIBRAW … YES`, `QuaZip … YES`.

## 5. Build

```bash
make -j$(sysctl -n hw.ncpu)
```

Quick smoke test (a window should open; Ctrl-C / quit to exit):

```bash
./nomacs.app/Contents/MacOS/nomacs    # logs to stdout
```

## 6. Extra image formats — build kimageformats (JXL, JP2, AVIF, JXR, Krita, …)

Homebrew has no `kimageformats` package, so nomacs builds it from source. ⚠️ The bundled
`build-kif.sh` hardcodes `cmake -G Ninja` and needs `CMAKE_PREFIX_PATH` exported, or it fails
with "unable to find Ninja" / "CMAKE_C_COMPILER not set":

```bash
CMAKE_PREFIX_PATH="/opt/homebrew/opt/qt6/lib/cmake:/opt/homebrew" \
  make kimageformats
```

It auto-`brew install`s the codec deps (libavif, jpeg-xl, jxrlib, karchive, …) and installs
`kimg_*.dylib` plugins into Qt's `imageformats` dir. HEIF is intentionally skipped (macOS/Qt
provide HEIC natively).

## 7. ⚠️ Gotcha — remove the crash-prone Qt jpeg2000 plugin (#1307)

Homebrew's jasper-based `libqjp2.dylib` crashes nomacs when opening `.jp2`. kimageformats'
OpenJPEG-based `kimg_jp2.dylib` (installed in step 6) replaces it. You must delete the dylib
**and** its CMake files together, or cmake later fails:

```bash
BK="$HOME/.nomacs-qt-jp2-backup"; mkdir -p "$BK"
QTPLUG=/opt/homebrew/opt/qt6/share/qt/plugins/imageformats
QTCMK=/opt/homebrew/opt/qt6/lib/cmake/Qt6Gui
mv "$QTPLUG/libqjp2.dylib" "$BK/" 2>/dev/null
mv "$QTCMK"/Qt6QJp2Plugin*.cmake "$BK/" 2>/dev/null
# restore later with: brew reinstall qtimageformats   (or move files back from $BK)
```

## 8. Register file types, then rebuild

This writes the supported-format list (now including JXL/JP2/AVIF/etc.) into `Info.plist`,
which is what enables Finder open-with / drag-and-drop.

```bash
# re-run configure first since step 7 removed the QJp2 cmake files
CMAKE_PREFIX_PATH="/opt/homebrew/opt/qt6/lib/cmake:/opt/homebrew/opt/opencv/lib/cmake/opencv4" \
  cmake -D ENABLE_QUAZIP=ON ../ImageLounge
make -j$(sysctl -n hw.ncpu)
make filetypes
make -j$(sysctl -n hw.ncpu)
```

> nomacs does **not** auto-register as the default app for any type. Set it per-type in Finder:
> **Get Info → Open With → nomacs → Change All**.

## 9. Build the self-contained bundle + dmg

```bash
make bundle    # runs macdeployqt; produces nomacs.app + nomacs.dmg
```

⚠️ macdeployqt prints `Cannot resolve rpath "@rpath/lib*Plugin.3.dylib"` for the optional
nomacs plugins — that is **non-fatal**, those plugins are copied in separately.

## 10. ⚠️ Gotcha — fix the OpenMP double-load crash (OMP Error #15), then re-sign

The bundle from step 9 **crashes on launch**:
`OMP: Error #15: Initializing libomp.dylib, but found libomp.dylib already initialized.`

Cause: macdeployqt leaves stale `/opt/homebrew/...` and build-dir `LC_RPATH`s on the OpenCV /
nomacsCore / plugin dylibs. So `@rpath/libopencv_core.413.dylib` resolves to the **Homebrew**
OpenCV, which drags in a *second* `openblas → libomp` stack → two OpenMP runtimes → abort.

Fix: strip every stale rpath, add bundle-relative `@loader_path`, then re-sign ad-hoc
(mandatory on arm64 — editing load commands invalidates the signature). Save this as
`fix-bundle.sh` and run it from the `build` dir:

```bash
#!/bin/bash
set -uo pipefail
APP="$(pwd)/nomacs.app"; cd "$APP/Contents"
BAD='^/opt/homebrew|^/Users/.*/WorkDir|/kimageformats/build|/build/libs$'
rpaths() { otool -l "$1" 2>/dev/null | awk '/ LC_RPATH$/{g=1} g&&/ path /{print $2; g=0}'; }

find . -type f | while IFS= read -r f; do
  file "$f" 2>/dev/null | grep -q Mach-O || continue
  case "$f" in
    ./MacOS/*)      want='@executable_path/../Frameworks' ;;
    ./Frameworks/*) want='@loader_path' ;;
    ./PlugIns/*)    want='@loader_path/../../Frameworks' ;;
    *)              want='@loader_path' ;;
  esac
  rpaths "$f" | grep -E "$BAD" | while IFS= read -r rp; do
    install_name_tool -delete_rpath "$rp" "$f" 2>/dev/null
  done
  rpaths "$f" | grep -Fxq "$want" || install_name_tool -add_rpath "$want" "$f" 2>/dev/null
done

# re-sign inside-out (ad-hoc)
find Frameworks PlugIns -name '*.dylib' -type f -exec codesign --force --sign - --timestamp=none {} \;
find Frameworks -maxdepth 1 -name '*.framework' -exec codesign --force --sign - --timestamp=none {} \;
cd "$APP/.."; codesign --force --sign - --timestamp=none nomacs.app
codesign --verify --deep nomacs.app && echo "signature OK"
```

```bash
bash fix-bundle.sh
```

Verify the crash is gone (should print exactly one libomp, from inside the bundle, and no
`/opt/homebrew` libs at runtime):

```bash
DYLD_PRINT_LIBRARIES=1 ./nomacs.app/Contents/MacOS/nomacs 2>&1 | grep -i libomp | sort -u
```

## 11. Install to /Applications

```bash
rm -rf /Applications/nomacs.app
cp -R nomacs.app /Applications/
open /Applications/nomacs.app      # Finder-style launch
```

Because the app is locally built it carries no quarantine attribute, so Gatekeeper opens it
normally even though it's ad-hoc signed (not notarized).

## 12. (optional) Rebuild a clean dmg from the fixed app

The `nomacs.dmg` produced in step 9 is from *before* the rpath fix; regenerate it:

```bash
STAGE=$(mktemp -d); cp -R nomacs.app "$STAGE/"; ln -s /Applications "$STAGE/Applications"
rm -f nomacs.dmg
hdiutil create -volname nomacs -srcfolder "$STAGE" -ov -format UDZO nomacs.dmg
rm -rf "$STAGE"
```

---

## Result

| Format source | Provides |
| ------------- | -------- |
| OpenCV + LibRaw | RAW (CR2/NEF/DNG…), high-quality thumbnails, adjustments |
| QuaZip | reading images inside `.zip` |
| Qt imageformats | TIFF, WEBP, ICNS, TGA, … |
| kimageformats | JPEG XL, JPEG 2000, AVIF, JPEG XR, EXR, Krita, OpenRaster, QOI, … |

[![nomacs-icon](https://nomacs.org/nomacs.svg)](https://nomacs.org)
