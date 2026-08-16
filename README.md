# OpenAqua

**Open-source documentation for iOS 27 MobileGestalt customization and PosterBoard exploit research.**

OpenAqua documents the exploits, tweaks, and detection mechanisms discovered while building Aqua — an iOS 27 MobileGestalt editor. This repository contains no source code. It is a reference for developers building similar tools.

---

## Table of Contents

1. [Exploits](#exploits)
2. [Tweaks](#tweaks)
3. [Hash Detection](#hash-detection)

---

## Exploits

### Overview

iOS 27 uses a sandbox system managed by `containermanagerd` (the ContainerManager daemon). Apps run in sandboxed containers and cannot access files outside their container without a sandbox extension. Two exploits allow bypassing this restriction:

1. **bad_query** — ContainerManager path traversal via `container_query` APIs
2. **.Trash symlink** — Sandbox bypass via `trashItemAtURL` through a symlink

---

### Exploit 1: bad_query (ContainerManager Path Traversal)

**Source:** [forcequitOS/bad_query](https://github.com/forcequitOS/bad_query)

**Supported versions:** iOS 26.0 – 26.6.1 / 27.0 beta 1–4

**Mechanism:**

The exploit uses ContainerManager's `container_query_*` APIs to obtain sandbox extensions for arbitrary file paths. The query traverses from a known container location to the target path using relative path traversal.

**Key APIs (from `libsystem_containermanager.dylib`):**

```c
container_query_create()                    // Create a new container query
container_query_set_class(query, class)     // Set container class
container_query_set_group_identifiers(query, identifier)  // Set group identifier
container_query_operation_set_flags(query, flags)          // Set operation flags
container_query_operation_set_part(query, part)            // Set traversal part
container_query_operation_set_part_domain(query, path)     // Set traversal path
container_query_get_single_result(query)    // Execute query
container_query_free(query)                 // Free query
container_copy_sandbox_token(result)        // Get sandbox token
sandbox_extension_consume(token)            // Consume sandbox extension
```

**Container Classes:**

| Class | Name | Purpose |
|-------|------|---------|
| 13 | `MCMSharedSystemDataContainer` | System groups (MobileGestalt) |
| 7 | `MCMSharedDataContainer` | App containers and shared data |

**Path Traversal:**

- **Class 13** (MobileGestalt): 8 levels of `../` from the cache directory
  - Cache path: `/var/containers/Shared/SystemGroup/systemgroup.com.apple.mobilegestaltcache/Library/Caches`
  - Traversal: `../../../../../../../..` (8 levels up to root)
  - Then append the absolute target path

- **Class 7** (App containers): 9 levels of `../` from the container root
  - Additional level needed to traverse from app container to root

**Operation Flags:**

```c
// Standard MobileGestalt access
0x0000008000000000ULL

// App Group access on iOS 26
0x0000000800000000ULL
```

**Critical discovery from godis:**

The `create` parameter in the C function skips the `lstat` existence check. Without this, the query fails with `-254` for paths that don't exist yet. Always pass `create: true` when accessing files that may not exist.

**Class 13 vs Class 7:**

- **Class 13** can reach ANY file on the system via path traversal from MobileGestalt's cache directory
- **Class 7** is used for writing to specific app containers (requires the container UUID as the group identifier)
- For **detection** of container UUIDs, use class 13 (no groupIdentifier) — it traverses from MobileGestalt to any path
- For **writing** to containers, use class 7 with the container UUID

---

### Exploit 2: .Trash Symlink (Sandbox Bypass)

**Source:** [nxtcoreee3/Bad-Poster](https://github.com/nxtcoreee3/Bad-Poster)

**Mechanism:**

Creates a symbolic link from `Documents/.Trash` to the target directory, then uses `trashItemAtURL` to write files through the symlink. The `.Trash` directory has special sandbox permissions that allow `trashItemAtURL` to follow symlinks.

**Steps:**

1. Remove any existing `.Trash` item in Documents
2. Create symlink: `Documents/.Trash` → target directory (e.g., PosterBoard descriptors path)
3. For each file to write:
   a. Copy the file to `Documents/{randomUUID}`
   b. Call `trashItemAtURL` on the copied file
   c. The trash operation follows the symlink, writing to the target directory
4. Remove the symlink

**Why raw `symlink()` doesn't work:**

The sandbox blocks raw `symlink()` calls. However, `FileManager.default.createSymbolicLink(at:withDestinationURL:)` uses a different code path that may bypass this restriction on certain iOS versions.

---

### Exploit 3: fsgetpath Enumeration

**Source:** [forcequitOS/bad_query](https://github.com/forcequitOS/bad_query)

**Mechanism:**

Uses the `fsgetpath` system call to enumerate directories by inode. This works even from within the sandbox because `fsgetpath` operates at the filesystem level, not the sandbox level.

**Usage:**

```c
fsgetpath(buf, buflen, &fsid, ino)
```

- Requires a valid `fsid_t` from `statfs()` on any existing path
- Enumerates inodes 1 through max (typically 2,000,000)
- Returns full paths for each inode
- Filter results by prefix to get children of a specific directory

---

## Tweaks

### Overview

MobileGestalt is a plist file that stores device capabilities and identity information. It is located at:

```
/var/containers/Shared/SystemGroup/systemgroup.com.apple.mobilegestaltcache/Library/Caches/com.apple.MobileGestalt.plist
```

The file contains:
- `CacheExtra` dictionary — most tweakable values
- `CacheData` binary blob — contains DeviceClassNumber and other encoded values
- Top-level keys — device identity and configuration

All tweaks below modify the `CacheData` and/or `CacheExtra` sections.

---

### Display & Appearance

| Tweak | Key(s) | Value | Effect |
|-------|--------|-------|--------|
| Enable Dynamic Island | `YlEtTtHlNesRBMal1CqRaA` | `1` | Enables Dynamic Island capability |
| Always-On Display | `2OOJf1VhaM7NxfRok3HbWQ`, `j8/Omm6s1lsmTDFsXjsBfA` | `1` | Enables AOD (may increase burn-in) |
| AOD Vibrancy | `ykpu7qyhqFweVMKtxNylWA` | `1` | Fixes AOD rendering issues |
| Disable Wallpaper Parallax | `UIParallaxCapability` | `0` | Stops wallpaper motion |
| Enable Liquid Glass LPM | `SAGvsp6O6kAQ4fEfDJpC4Q` | `1` | Low-performance mode for Liquid Glass |
| Disable Liquid Glass LPM | `SAGvsp6O6kAQ4fEfDJpC4Q` | `0` | Mutually exclusive with above |

### Hardware Capabilities

| Tweak | Key(s) | Value | Effect |
|-------|--------|-------|--------|
| Boot & Shutdown Chime | `QHxt+hGLaBPbQJbXiUJX3w` | `1` | Enables boot chime |
| Charge Limit Menu | `37NVydb//GP/GrhuTN+exg` | `1` | Shows charge limit settings |
| Tap to Wake | `yZf3GTRMGTuwSV/lD7Cagw` | `1` | For iPhone SE and similar |
| Camera Control | `CwvKxM2cEogD3p+HYgaW0Q`, `oOV1jhJbdV3AddkcCg0AEA` | `1` | iPhone 16 Camera Control settings |
| Apple Pencil | `yhHcB0iH0d1XzPO/CFd3ow` | `1` | Shows Apple Pencil settings |
| Action Button | `cT44WE1EohiwRzhsZ8xEsw` | `1` | Shows Action Button settings |
| Collision SOS | `HCzWusHQwZDea6nNhaKndw` | `1` | Shows collision detection in SOS |
| Pulse Width Modulation | `6IejgN+1Fmu5/QrZFOIeNw` | `1` | Enables PWM display feature |

### iPad Capabilities

| Tweak | Key(s) | Value | Effect |
|-------|--------|-------|--------|
| Stage Manager | `qeaj75wk3HF4DwQ8qbIi7g` | `1` | Enables Stage Manager support |
| Allow iPad Apps | `9MZ5AdH43csAUajl/dU+IQ` | `[1, 2]` | Enables iPad app compatibility |
| Enable iPadOS Mode | Multiple | — | See below |

**iPadOS Mode** requires both `CacheExtra` and `CacheData` modifications:

CacheExtra keys:
- `mG0AnH/Vy1veoqoLRAIgTA` = 1 (Medusa Floating Live Apps)
- `UCG5MkVahJxG1YULbbd5Bg` = 1 (Medusa Overlay Apps)
- `ZYqko/XM5zD3XBfN5RmaXA` = 1 (Medusa Pinned Apps)
- `nVh/gwNpy7Jv1NOk00CMrw` = 1 (Medusa PIP)
- `uKc7FPnEO++lVhHWHFlGbQ` = 1 (Device is iPad)

CacheData modification:
- Locate `DeviceClassNumber` offset in the binary blob
- Change value from `1` (iPhone) to `3` (iPad)
- Offset is found via regex pattern matching on the hex-encoded CacheData

### Internal & Research

| Tweak | Key(s) | Value | Effect |
|-------|--------|-------|--------|
| Apple Internal Install | `EqrsVvjcYDdxHBiQmGhAWw` | `1` | Enables Metal HUD and internal features |
| Internal Storage View | `LBJfwOEzExRxzlAnSuI7eg` | `1` | Shows internal files in Storage settings |
| Security Research Device | `XYlJKKkj2hztRP1NWWnhlw` | `1` | Marks device as SRD |
| Apple Intelligence | `A62OafQ85EJAiiqKn4agtg` | `1` | Enables Apple Intelligence (may need device spoofing) |

### Additional Features

| Tweak | Key(s) | Value | Effect |
|-------|--------|-------|--------|
| Pulse Width Modulation | `6IejgN+1Fmu5/QrZFOIeNw` | `1` | Enables PWM display feature |
| Metal HUD in All Apps | `EqrsVvjcYDdxHBiQmGhAWw` | `1` | Shows GPU performance overlay (same as Internal Install) |

### Device Subtype (Dynamic Island)

The Dynamic Island is controlled by the `ArtworkDeviceSubType` subkey within the `oPeik/9e8lQWMszEjbPzng` dictionary in `CacheExtra`:

| Subtype | Device |
|---------|--------|
| 2436 | iPhone X Gestures (SE) |
| 2556 | iPhone 14 Pro |
| 2796 | iPhone 14 Pro Max |
| 2622 | iPhone 16 Pro |
| 2868 | iPhone 16 Pro Max |
| 2736 | iPhone Air |

When changing the subtype, also set `YlEtTtHlNesRBMal1CqRaA` = 1 (Dynamic Island capability).

### Model Name

The device model name displayed in Settings > About is stored in `oPeik/9e8lQWMszEjbPzng` > `ArtworkDeviceProductDescription` in `CacheExtra`.

### Siri AI Region

To enable Siri AI on unsupported devices:

1. Set `A62OafQ85EJAiiqKn4agtg` = 1 (AI eligibility)
2. Spoof product type: `h9jDsbgj7xIVeIQ8S3/X3Q` (e.g., `iPhone16,1` for iPhone 15 Pro)
3. Spoof hardware: `oYicEKzVTz4/CxxE05pEgQ` (e.g., `D83AP`)
4. Spoof CPU: `5pYKlGnYYBzGvAlIU8RjEQ` (e.g., `t8130`)
5. Set region: `h63QSdBCiT/z0WU6rdQv6Q` = "LL", `yK+xavymRGZ3xWc1tb8XDg` = "LL/A"
6. Set regulatory model: `97JDvERpVwO+GHtthIh7hA` (e.g., "A2848")

---

## Hash Detection

### Overview

PosterBoard (the lock screen wallpaper system) stores its data in an app container at:

```
/var/mobile/Containers/Data/Application/{UUID}/
```

To write wallpapers to PosterBoard, you need to find this UUID (called the "app hash"). This documentation explains how to detect it automatically.

### Container Enumeration

iOS containers are stored in:
- `/var/mobile/Containers/Data/Application/` — Regular app containers
- `/var/mobile/Containers/Data/InternalDaemon/` — Internal daemon containers
- `/var/mobile/Containers/Data/PluginKitPlugin/` — Plugin containers

Each container is a UUID-named directory. To find PosterBoard, enumerate these directories and read their metadata.

### Metadata Plists

Each container has a metadata plist at its root:
- `.com.apple.mobile_container_manager.metadata.plist` (hidden, with dot prefix)
- `com.apple.mobile_container_manager.metadata.plist` (without dot prefix)

The plist contains `MCMMetadataIdentifier` — the bundle ID of the app that owns the container.

### Detection Algorithm

1. **Enumerate directories** using `fsgetpath` (from bad_query) on each container path:
   - `/var/mobile/Containers/Data/Application`
   - `/var/mobile/Containers/Data/InternalDaemon`
   - `/var/mobile/Containers/Data/PluginKitPlugin`

2. **For each UUID directory**, try to read its metadata plist using bad_query with class 13 (no groupIdentifier):
   ```
   BadQueryConsumeForPath(metadataPath, create: true, groupIdentifier: nil)
   ```
   This uses class 13 to traverse from MobileGestalt's cache to any file on the system.

3. **Read the metadata** and check if `MCMMetadataIdentifier` equals `com.apple.PosterBoard`.

4. **Return the UUID** if found.

### Why Class 13 Works for Detection

Class 13 (MobileGestalt SystemGroup) with no groupIdentifier traverses from MobileGestalt's cache directory (`/var/containers/Shared/SystemGroup/systemgroup.com.apple.mobilegestaltcache/Library/Caches`) up 8 levels to root, then to any target path. This can reach ANY file on the system, including app container metadata.

### Why Class 7 Doesn't Work for Detection

Class 7 (SharedDataContainer) requires the container UUID as the group identifier — but you don't know the UUID yet (that's what you're trying to find). Chicken-and-egg problem.

### PosterBoard Container Path

Once you have the UUID, PosterBoard's wallpaper data is at:
```
/var/mobile/Containers/Data/Application/{UUID}/Library/Application Support/PRBPosterExtensionDataStore/61/Extensions/{extensionID}/descriptors/
```

Where `extensionID` is typically `com.apple.WallpaperKit.CollectionsPoster` for lock screen wallpapers.

---

## Contributing

This is a documentation-only repository. If you find new tweaks, exploits, or detection methods, please open an issue or pull request with the findings.

## License

MIT — use this information however you want.
