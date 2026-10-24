# Introduction

This repository contains a collection of resources for WinlatorXR, including .wcp packages, preinstalled containers, and ready-to-use runtimes. These resources are designed to simplify setup and help users get started with running Windows applications and games on standalone VR headsets using WinlatorXR.

# Packaging OXRWXR and OpenComposite as WCPs

The PC VR shortcut option uses two runtimes, copied into each container at launch by
`PcvrRuntime.installPayload` (`app/src/main/java/com/winlator/xr/utils/PcvrRuntime.java`):

| Runtime | Content type | Container folder | Files |
|---|---|---|---|
| OXRWXR (OpenXR) | `OXRWXR` | `C:\oxrwxr\` | `openxr_wxr.dll`, `openxr_wxr32.dll` |
| OpenComposite (OpenVR) | `OpenComposite` | `C:\OpenComposite\Runtime\bin\` | `vrclient_x64.dll`, `vrclient.dll` |

For each type the **installed WCP with the highest `versionCode` wins**. The copy bundled in
the APK (`app/src/main/assets/pcvr/`) is only used when no WCP of that type is installed. So a
newer runtime bundled in the APK does **not** reach a container while an older WCP is still
installed: ship a higher `versionCode` WCP, or uninstall the old one.

The container is refreshed whenever the bundled `version.txt` or the chosen WCP entry name
changes, so installing a new WCP is enough; no container reset is needed.

## Version naming

| Field | OXRWXR | OpenComposite |
|---|---|---|
| `versionName` | `OXRWXR_v<N>` | `OpenComposite_v<N>` |
| `versionCode` | `<N>` | `<N>` |
| File name | `OXRWXR_v<N>.wcp` | `OpenComposite_v<N>.wcp` |

Rules:

- **`<N>` goes up by one for every build handed out**, even a rebuild of the same source.
  `versionCode` and the number in `versionName` must always match.
- **Use an underscore, never a dash, in `versionName`.** The installed folder is named
  `<versionName>-<versionCode>` and is split on dashes, so a dash in the name breaks lookup.
- **Put the product name in `versionName`.** The Contents screen shows only
  "Version: …" and "Version code: …", with no type, and lists OXRWXR, OpenComposite, Goldberg
  and the runtime together, so a bare git sha tells the user nothing.
- **Put the source in `description`**: branch, commit sha, and what changed since the
  previous number. It is shown in the content info dialog and is the only record of what a
  number contains. Say so if the build includes uncommitted changes or needs a matching app
  build.
- Never reuse a number, including for a build that was dropped. Find the current highest
  number from the last WCP handed out, not from the bundled APK copy.

When the APK's bundled copy is updated, record it in `app/src/main/assets/pcvr/version.txt`
(`oxrwxr=<sha> oxrwxr32=<sha>-x86 opencomposite=<sha>`) and name the WCP versions in the
commit message, e.g. "update included PCVR runtimes to OXRWXR v23 and OpenComposite v14".

## 1. Build the DLLs

Both need Visual Studio 2022 (MSVC x64/x86), CMake and the Vulkan SDK (`VULKAN_SDK` set).
Build in a short path; a deep path (such as a temp folder) makes MSBuild fail with `FTK1011`.

### OXRWXR (`OpenXR-WXR-private`)

Always `git fetch` and branch from `origin/master`. A runtime built from a stale branch loads
and runs but leaves the headset black, which looks exactly like a code regression.

```bat
build.bat      :: 64-bit, output oxrwxr\openxr_wxr.dll
build32.bat    :: 32-bit, output oxrwxr\openxr_wxr32.dll
```

`build.bat` also runs `adb push` to `/sdcard/Download/Winlator`. That folder is not what the
runtime loads, so ignore it; the WCP is the delivery route. The six C4311/C4312
pointer-truncation warnings are pre-existing.

### OpenComposite (`OpenComposite-private`)

Needs its submodules (`git submodule update --init --recursive`); a fresh `git worktree`
has none, so build in the main checkout.

```bat
cmake -S . -B build/64bit -G "Visual Studio 17 2022" -A x64
cmake --build build/64bit --config Release
cmake -S . -B build/32bit -G "Visual Studio 17 2022" -A Win32
cmake --build build/32bit --config Release
```

Outputs: `build/64bit/bin/Release/vrclient_x64.dll` and `build/32bit/bin/Release/vrclient.dll`.
The sources are CRLF; keep that when editing.

## 2. Stage the files

Make one folder per version holding the DLLs and a `profile.json`, all **flat at the root**
(no subfolder):

```
OXRWXR_v24/
  profile.json
  openxr_wxr.dll
  openxr_wxr32.dll
```

OXRWXR `profile.json`:

```json
{
  "type": "OXRWXR",
  "versionName": "OXRWXR_v24",
  "versionCode": 24,
  "description": "OXRWXR OpenXR runtime for the PC VR shortcut option, 64-bit and 32-bit. Built from master <sha>: <what changed since v23>.",
  "files": [{"source":"openxr_wxr.dll","target":"openxr_wxr.dll"},{"source":"openxr_wxr32.dll","target":"openxr_wxr32.dll"}]
}
```

OpenComposite `profile.json`:

```json
{
  "type": "OpenComposite",
  "versionName": "OpenComposite_v15",
  "versionCode": 15,
  "description": "OpenComposite OpenVR runtime for the PC VR shortcut option, 64-bit and 32-bit. Built from <branch> <sha>: <what changed since v14>.",
  "files": [{"source":"vrclient_x64.dll","target":"vrclient_x64.dll"},{"source":"vrclient.dll","target":"vrclient.dll"}]
}
```

`type` must be exactly `OXRWXR` or `OpenComposite`. Each `target` is a plain file name inside
the container folder in the table above; empty targets and `..` are rejected on install.
Always ship both DLLs. Only `openxr_wxr32.dll` falls back to the APK copy; any other file
missing from an installed WCP is not copied at all, leaving whatever older copy the container had.

## 3. Pack

A WCP is a tar archive compressed with **zstd** (xz also installs, but zstd is the convention).
On Windows use the system `tar.exe`, which is bsdtar with zstd built in:

```sh
/c/Windows/System32/tar.exe -c --zstd --options zstd:compression-level=19 \
    -f OXRWXR_v24.wcp -C OXRWXR_v24 profile.json openxr_wxr.dll openxr_wxr32.dll
```

Git Bash's `tar` is GNU tar, which needs a separate `zstd` binary and fails without one. That
does not mean zstd is unavailable; call the system `tar.exe` instead. 7-Zip can read zstd but
cannot write it.

Check the result lists the files at the root and the profile reads back:

```sh
/c/Windows/System32/tar.exe -tvf OXRWXR_v24.wcp
/c/Windows/System32/tar.exe -xOf OXRWXR_v24.wcp profile.json
```

## 4. Install and test

```sh
adb push OXRWXR_v24.wcp /sdcard/Download/
```

Install it from the app's Contents screen, then launch a PC VR shortcut. Recheck a known-good
title after every change (for example Beat Saber for OXRWXR, and an OpenComposite title) as
well as the title the change was for.

To confirm what a container is running, the installed copies are under
`/data/data/com.winlator.cmod/files/imagefs/home/xuser-<N>/.wine/drive_c/` (`oxrwxr/` and
`OpenComposite/Runtime/bin/`), readable with `adb shell run-as com.winlator.cmod`.
`oxrwxr/.winlatorxr_payload_version` names the WCP entries last copied in.

## Proton WCP (for reference)

The Proton package that carries `wxr_bridge.dll` / `wxr_bridge.so` is a different kind of WCP:
it uses a `wine` block instead of a `files` list, keeps its existing `versionName`
(`11.0-2-arm64ec`, dashes included) and only raises `versionCode`, because containers store
the Wine version by entry name and renaming it would orphan them. Never extract it on Windows
to repack it: it contains symlinks that Windows tar cannot recreate. Replace members in the
tar stream instead and recompress with the system `tar.exe`.
