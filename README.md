# Portable-Flatpak
Portable Flatpak logic that encapsulates app execution, user data, overrides, icons, and optional binaries inside a single movable folder. Copy, rename, and run Flatpaks anywhere using a auto-detection, zero external config wrapper— Windows/Mac-like portability on Linux.

# Practical Pattern for Portable Flatpaks

This document describes a practical approach to achieving easier **portability** for applications on Linux — specifically with Flatpaks — by encapsulating the application, execution logic and persistent user data inside a single, movable directory.

> **Goal:** transferring an application and its configuration between PCs should be as simple as copying a folder from a desktop to a laptop and running the program instantly.
---

## 1. Summary

The current concept of “portable software” in the Linux ecosystem is fragmented. Formats like AppImage offer single-file portability, but they typically do **not** encapsulate *user data* (configurations, saves, caches) inside the same container, which leads to data fragmentation.

This approach provides a pragmatic mechanism to encapsulate both application execution logic and persistent user data — and optionally binaries — into a single movable directory. The folder is driven by an embedded master logic that acts as middleware between the user and Flatpak. The launcher detects location and names and launches the application.

This is a tooling-first, practical example intended to work today rather than a formal standard, but it can inform future Flatpak improvements designed with portability and flexibility in mind so
expect inconsistencies
---

## 2. Problem: Flatpak Limitation

with Flatpak, the portable model is constrained by rigid file paths:

1. **Binary location:** Applications are installed to `/var/lib/flatpak` (system) or `~/.local/share/flatpak` (user).
2. **Data location:** User data is isolated under `~/.var/app/`.

**Consequences:**

* Users cannot easily move an installation to a secondary drive to free root partition space.
* Formatting the root partition results in data loss. Backing up `~/.var` entirely can be redundant — not all apps are important.
* Manually selecting which `~/.var/app` subfolders to back up is annoying.
* Syncing application state between machines requires managing multiple directories.

These constraints motivated the practical approach described below.

---

## 3. My Solution: The ideal portable application

Use a flexible logic that acts as middleware between the user and Package, on this case Flatpak. your one click* `runner` is a engine: it detects its own location, handles names, new locations, detects use cases and launches the application. you can name the `runner` whatever you want

### 3.1 Core Philosophy

* **Adaptable:** Users should be able to rename and move most directories freely. The system should detect the folder's current absolute path and still work when the folder moves directories or names.
* **No metadata files:** configuration internally to avoid extra files and clutter.
* **Expressive layout:** Users should be free to organize the portable directory to their taste; a sensible default structure should exist.

This is a tooling-first approach intended to work now rather than a formal standard.

### 3.2 Possibles Directories Structures

You should be able to have a flexible structure that preserves your needs, with the freedom to name everything whatever you want

```
$PORTABLE_ROOT/                    # renameable
├── runner                          # one-click run application (Entry Point) # renameable
├── stuff                           # (Optional) anything the user would want to keep; should never break anything # renameable
├── binaries                        # (Optional) Offloaded Flatpak binaries for ~/.local/share/flatpak/app/$APPID # renameable
└── data                            # (On by default) Persistent User Data for ~/.var/app/$APPID  # renameable
```

or, in a minimal layout:

```
$PORTABLE_ROOT/                    # RENAMABLE
├── runner                          # one-click run application (Entry Point) # RENAMABLE
├── stuff                           # (Optional) anything the user would want to keep; should never break anything # RENAMABLE
└── $APPID/                        # (On by default) Persistent User Data # RENAMABLE
```
if you want to go even more minimal
```
$PORTABLE_ROOT/                    # RENAMABLE
├── Brave                           # AppID=com.brave.Browser one-click run application (Entry Point) # RENAMABLE
├── Discord                         # AppID=com.discordapp.Discord one-click run application (Entry Point) # RENAMABLE
├── Steam                           # AppID=com.valvesoftware.Steam one-click run application (Entry Point) # RENAMABLE
```
this effectively work as a program that just installs flatpak, adds flathub, installs the application by its $APPID and run your program with one click

### 3.3 limitations

**Hard Constraints:**
* The user should never rename or move the folder while the app is running.

**soft Constraints:**
* `"$DATA"` folders cannot be searched more than two directories deep. (e.g., `$PORTABLE_ROOT/things/stuff/$DATA`) will not be searched will not be found
* naturally avoid ntfs or others unsupported file systems, sending
```
notify-send "Error" "File system unsupported, Move into a linux File system drive\!" -u critical
```
(optional, but no ntfs by default).


```
├────────── $PORTABLE_ROOT
├── cache
├── config
├── current -> x86_64/stable
├── data
├── .ld.so
├── .local
├── runner
└── x86_64
8 directories, 1 file
```
* this setup could also be a theoretically predictable pattern, but at least for me the complexity increases exponentially.
* so i wont include surgically individually detecting each sub-directory on my implementation
 
 
 
### 3.3 Detection Mechanism

**user A — Data Persistence**
* **Trigger:** Presence of `"$DATA"` and absence of `"$BIN"` 
* **results:** Keep only user saves/configuration portable; binaries remain on the host. Binaries are retrievable from Flathub.

**user A.2 — Binary Offloading**
* **Trigger:** Presence of `"$DATA"` And presence `"$BIN"` 
* **results:** Store both binaries and user data on the portable directory.

**user C — Click and run**
* **Trigger:** absence of `"$DATA"` And absence of `"$BIN"` 
* **results:** effectively just installs the application and executes it

### 3.4 Declaring `"$DATA"` and `"$BIN"` though its contents
```
├────────── $PORTABLE_ROOT
├── перевод       # `"$DATA"`
│   ├──  cache
│   ├── .cache
│   ├── .config
│   ├──  config
│   ├──  data
│   ├── .ld.so
│   ├── .local
│   ├── .nv
│   ├── .pki
│   └── .var
└── runner                          # one-click run application (Entry Point) # renameable
8 directories, 1 file
```
all flatpaks `"$DATA"`  directories, at some level, have some of these files, so we can confirm `$DATA` directory by checking if its contents have 3 or more list:
(`cache`, `.cache`, `.config`, `config`, `data`, `.ld.so`, `.local`, `.nv`, `.pki`, `.var`)

```
├────────── $PORTABLE_ROOT
├── двоичный       # "$BIN" 
│   ├── current -> x86_64/stable
│   └── x86_64
└── runner                          # one-click run application (Entry Point) # renameable
└── перевод
```
alternatively  flatpaks always include on `"$BIN"`, Directories a:
(`x86_64`) folder
and a: 
(`current`) -> (`x86_64/stable`)
symbolic link: 
so if a directory contains those two we can confirm its it is the `"$BIN"` directory

## 3.5 contents aware system. No configuration,

*   The system should detect with no extra configuration needed, what the user wants to keep portable. through the presence or absence of directories or files, so users don’t need to edit config files or use the terminal.

```
├────────── $PORTABLE_ROOT
├── program
│   ├── more program stuff
│   │   ├── current -> x86_64/stable
│   │   └── x86_64
│   └── sister draws
├── informations
│   ├── anything
│   └── eerm
│       ├── cache
│       ├── config
│       ├── data
│       ├── .ld.so
│       └── .local
└── runner                 # one-click run application (Entry Point) # renameable
```
even this crazy setup, the system should be able to detect `eerm="$DATA"` and `program stuff="$BIN"`

now. if directories are empty `"$DATA"` and `"$BIN"`  couln't be determined by its contents using the method above, we proceds with:**

### 3.6 language assumptions

*  help you to install the application, and the system know what are the `"$DATA"`  and `"$BIN"`  directory locations.
*  language assumptions are on by default. directories named (`dat`), (`data`) or (`dados`) are detected as the `"$DATA"`    
* (`bin`) (`binaries`) (`binarios`) as `"$BIN"`
*  you should be free to control assumptions, by creating your own list or disabling completely
*  of more than one of those those for type,for `"$BIN"` or for `"$DATA"` exist, chose on alphabetic order the first directorie that is empty
*  (e.g., `$PORTABLE_ROOT/bin/dat`) is naturally invalid, (e.g., `$PORTABLE_ROOT/$DATA/"$BIN"`) also is invalid. abort.
*  (e.g., `$PORTABLE_ROOT/binaries/bin`) should validade the last  and not fire too early. aware of (e.g., `$PORTABLE_ROOT/$BIN/"$BIN"`) validade the last one.
*  language assumptions should only be used if Declaring `"$DATA"` and `"$BIN"` through its contents, failed

## 4. My Stretch: Desktop Integration

* naturally, installing flatpaks adds a desktop entry on your Desktop environment, we want to preserve this behavior by default. 
* you should be able to disable desktop integration. so you can only run the application by clicking the `runner`

*automatically applies a custom icon, by detecting by the presence of any images inside the `$PORTABLE_ROOT` so you don’t have to edit anything manually

```
├────────── $PORTABLE_ROOT
├── runner
├── stuff
├── icon        #lets supposed its a svg, but is has no extension, should work without any problem, **never depend exclusively file extension, .png .svg. ico. .jpg .jpeg or .webp
├── binaries
└── data
```

by dragging any image on `$PORTABLE_ROOT` we should automatically apply by editing the `icon=` line on the desktop file, specifically for flatpaks:

system wide
 `/var/lib/flatpak/exports/share/applications/$APPID`
or for a --user install at
 `~/.local/share/flatpak/exports/share/applications/$APPID`


 * users may also want to keep custom .desktop file.
 * like everything here we should automatically apply it by its presence.
 * if you don't want a desktop file, just don't have any it on `$PORTABLE_ROOT`
```
├────────── $PORTABLE_ROOT
├── runner
├── stuff
├── icon.png       #lets say its a png, should work with any iamge format, never depend exclusively on detecting the file extension, .png .svg. ico. .jpg .jpeg or .webp
├── custom.desktop
├── binaries/
└── data/
```
  * if a desktop file is already included on the $PORTABLE_ROOT just edit the `icon=` on this desktop file so.
  * no need to touch  `~/.local/share/flatpak/exports/share/applications/$APPID` or  `/var/lib/flatpak/exports/share/applications/$APPID`
## 4.1 Portable folder icon integration

The portable directory should visually represent the application by automatically applying an icon to the **folder itself** "$PORTABLE_ROOT".

### Behavior

At launch or initialization time, the runner selects a single image to represent the portable folder:

* **Primary source:** any image located at the root of `$PORTABLE_ROOT`
* **Fallback source:** the application icon exported by the Flatpak runtime at
  `/$BIN/current/active/export/share/icons/hicolor/scalable/apps/$APPID.svg`

The selected image is resolved to an absolute path. The icon must remain valid after the portable directory is relocated to a different mount point or system.

### Nautilus integration model

GNOME Files (Nautilus) supports per-directory custom icons through filesystem metadata. The runner applies the resolved image path as the directory’s custom icon so the folder consistently appears with the application icon in Nautilus views (grid, list, sidebar, recent locations).

The icon assignment is metadata-based and does not modify the directory contents. Clearing this metadata restores the default folder appearance.

### Portability constraints
* Absolute paths must be recalculated when the portable directory is moved.
* Icon metadata may be cached by Nautilus; visual updates are not guaranteed to be immediate.


### Automating permissions (optional, on by default)

At launch time the runner evaluates the override file state:

* **If `~/.local/share/flatpak/overrides/$APPID` does not exist or is empty**

  * The runner writes the portable override into this location (if one is defined internally).
  * This enables the required permissions for the portable app.

* **If `~/.local/share/flatpak/overrides/$APPID` exists and is not empty**

  * The runner reads it into memory.
  * If the contents differ from the internally defined override, the runner prefers the existing user override and applies in-memory normalization only (see below).
  * No file is duplicated or stored elsewhere.

* **override files should be stored internally inside the own `runner`. so no annoying extra files or hidden files.**
* **(e.g., so no "$PORTABLE_ROOT/.overrides/$APPID" this would be stupid)

#### Path and username normalization

For Migrating for distros like fedora that use /run/media or on a system with a new user name

* Convert usernames and media mount inside paths:

```
/media/burtjo/... → /run/media/jason/...
```

**Example:**

```
[Context]
devices=dri
filesystems=/media/burtjo/SSD/SteamLIB;/media/burtjo/SSDpkg/Gaming/Steam
```

becomes

```
[Context]
devices=dri
filesystems=/run/media/jason/SSD/SteamLIB;/run/media/jason/SSDpkg/Gaming/Steam
```

The resulting content is written directly to
`~/.local/share/flatpak/overrides/$APPID`.


### 5 Symlink Trick

 * Flatpak looks for user data in
 `~/.var/app/$APPID`


 *and user install binaries at
 `~/.local/share/flatpak/app/$APPID`
  or system wide at
 `/var/lib/flatpak/app/$APPID`

* Using Symlinks to achieve Portable applications, Specifically Flatpaks by redirecting that location to the portable folder — without patching Flatpak.
* This Symlinks method can be used for other package formats, but complexity can increase.

### 6 My practical execution logic

0. **on your one click `run` the system should:**

1. **automatically Installs Flatpak and adds flathub to your distro** # for flatpaks

2. **computes its current absolute path for** `$PORTABLE_ROOT` **example** `/media/user/USB/Games/Osu`

3. **defines the** `"$DATA"` **and** `"$BIN"` **directories using the flatpaks contents aware method, or the language assumption if contents aware fail.**

4. **defines a valid flathub** `$APPID`**:** 
   * we need to define a `[VALID_APPID]` a constant and unique identifier of the application consisting of the reverse-DNS format {tld}.{vendor}.{product}.
   
5. **internal** `$APPID`**:** 
```
├────────── $PORTABLE_ROOT
├── runner         #AppID=$APPID
├── $BIN
└── $DATA
```
   * no extra files, `$APPID` its stored internally.
   * check if `$BIN/current/active/export/bin/$APPID` exist, this location always has a file with the a `[VALID_APPID]` `$APPID`
   * `$BIN/current/active/export/bin/$APPID` is supposed to always pass for a valid app id
   * checking if the `$BIN/current/active/export/bin/$APPID` is `[VALID_APPID]` is usefull for debuging the `[VALID_APPID]` detection logic
   * check if if `runner` name is a `[VALID_APPID]` and if is conflicting with the`$BIN/current/active/export/bin/$APPID`
# the user can rename the `runner` for anything, including a flathub `$APPID` so is a easy way for us define the `$APPID`,
   * if the `runner` file name is conflicting, rename the `runner` to the `$APPID` at `$BIN/current/active/export/bin/$APPID`
   * as soon as a `[VALID_APPID]` is resolved, the internal `AppID=0` is overwritten to `AppID=$APPID`
   * if `$BIN/current/active/export/bin/$APPID` dont exist. there are others ways to check for a `$APPID`:
   * if internal `#AppID=0` and the `runner` file name. if its a `[VALID_APPID]` 
   * if `$BIN/current/active/export/bin/$APPID` dont exist
   * and the `$APPID` was a successfully found `[VALID_APPID]` though the `runner` name, the stored `$APPID` as, for example `AppID=0` will be overwritten to `AppID=$APPID`

5. **internal** `$APPID`**:** 
```
├────────── $PORTABLE_ROOT
├── $APPID     #AppID=$nonMatchingAPPID
├── $BIN
└── $DATA
```

   * if the `runner` name is a `$APPID` but its a different one from the internal, `runner` name is at a higher priority, it overwrites the internal app,
   * expect if its conflicting with the `$BIN/current/active/export/bin/$APPID`
   * if the `"$BIN"` was successfully found using the flatpaks contents aware method. 
```
├────────── $PossibleAPPID
├── $PossibleAPPID
│   ├── $PossibleAPPID
│   │   ├── current -> x86_64/stable
│   │   └── x86_64
│   └── sister draws
├── $PossibleAPPID
│   ├── anything
│   └── $PossibleAPPID
│       ├── cache
│       ├── config
│       ├── data
│       ├── .ld.so
│       └── .local
└── $PossibleAPPID       # runner here, AppID=0, by default (cache should be self contained, no extra files)
```
here are example for directories we check for a possible `$APPID`

   * $APPID, `"$DATA"` and `"$BIN"`  folders cannot be searched more than two directories deep (e.g., `[Portable Path]/[things]/[stuff]/$APPID` is invalid by default).
   * aware of examples like org.telegram.desktop, .desktop is not a extension, its part of the $APPID,
   * aware of examples like `org.telegram.desktop.sh` or `org.telegram.desktop.py`, or even just `org.telegram.desktop` without any extension.
```
.
├── $PossibleAPPID
│   ├── $PossibleAPPID
│   │   ├── current -> x86_64/stable
│   │   └── x86_64
│   └── sister draws
├── $PossibleAPPID
│   ├── anything
│   └── $PossibleAPPID
│       ├── cache
│       ├── config
│       ├── data
│       ├── .ld.so
│       └── .local
└── $PossibleAPPID        # runner here, another $APPID is cached internally, like AppID=$APPID by default it should be AppID=0 (cache should be self contained, no extra files)

```
w 

7. **Definitions:**
   
   * after all of this, only after the `$APPID` is finally detected,
   * Defines its own absoulte path as variable for `$PORTABLE_ROOT`
   
   * Define `[USER_MODE]` for user install (default). or `[SYSTEM_MODE]` for system wide install
     user mode at
     `$BIN_FLATPAK` = `~/.local/share/flatpak`. or for system wide `$BIN_FLATPAK` = `/var/lib/flatpak`.
     or for system wide at:
   * Defines `$BIN_FLATPAK` variable if choosen user mode or system wide:

   * defines $BIN_TARGET for `$BIN_FLATPAK/app/$APPID`
   * define if `[DESKTOP_FILE]` exist at $PORTABLE_ROOT , can be defined at any plain at the text file with a header "[Desktop Entry]"
   * if a desktop file is present on the $PORTABLE_ROOT define $PORTABLE_ROOT/[DESKTOP_FILE] For $DESKTOP_FILE if a desktop file is present on the $PORTABLE_ROOT
   * if no `[DESKTOP_FILE]` is present on the $PORTABLE_ROOT
   * define the `$DESKTOP_FILE` AS `$BIN_FLATPAK/exports/share/applications/$APPID.desktop`.




8. **if Its alredy installed:**

   * if it already is installed, compare how its installed with how its the `[Portable Path]` configuration. determined by the layout previously.
   * If `~/.var/app/$APPID` is a broken symlink (from a previous location), remove it.
   * if `~/.var/app/$APPID` its not a symlink and the user language assumption-
   * check if the detected `"$DATA"` from LANGAGE_INFERRING_`"$DATA"` directories are empty
   * if they are, move the data from `~/.var/app/$APPID/` to the LANGAGE_INFERRING_`"$DATA"` directory,
   * be aware of `data/$APPID` `dados/$APPID` `dat/$APPID` setups as well,
   * and If `~/.var/app/$APPID` is a standard directory, rename it by appending a rotation suffix (e.g., `com.vendor.org` → `com.vendor.org1`) so the original is backed up. If `com.vendor.org1` already exists, rotate to `com.vendor.org2`, and so on
   * If `$BIN_TARGET` is a broken symlink (from a previous location), remove it.
   * If `$BIN_TARGET` is a functional symlink, delete it to prepare for a new link.


9. **if its already installed at a oposing mode(sytem wide or user) other way around aware**

   * uninstall and reinstall to the right mode by default
   * i want to also experiment with
   * if `/var/lib/flatpak/app/$APPID`is not a broken symlink and it is a working directory or a working symlink,
   * wipe internal ``"$BIN"` ` and move `$BIN_TARGET/x86_64` to `$BIN/` then symlink ``"$BIN"` /x86_64/stable` to ``"$BIN"` /current`
   * if its not using a [LOCAL_DESKTOP_FILE] and DESKTOP_INTEGRATION=1
   * move `/var/lib/flatpak/app/$APPID/current/active/export/share/applications/$APPID` to `~/.local/share/flatpak/app/$APPID/current/active/export/share/applications/$APPID`
   * and symlink `~/.local/share/flatpak/app/$APPID/current/active/export/share/applications/$APPID` `~/.local/share/flatpak/exports/share/applications/$APPID`
   * or if its using a [LOCAL_DESKTOP_FILE] or DESKTOP_INTEGRATION=0
   * proceeds with the symlink

10. **Linking**
   * dinamically symlink what is needed based on the configuration of the directories layout
   * Symlink ``"$DATA"`` to `~/.var/app/$APPID`
   * Symlink ``"$BIN"` ` to `$BIN_TARGET`
   * Symlink `[LOCAL_DESKTOP_FILE]` to `[DESKTOP_FILE]`

11. **if its not already installed**
   * install according to configuration
   * installs the application normally, flatpak install --user [APPID]
   * or system wide if was configured that way

12. **icon phase:**
   * if [ICON] exist
   * change the `[DESKTOP_FILE]` or `[LOCAL_DESKTOP_FILE]` (if it exists) `icon=` line to include the `[ICON]` absolute path


13. **Execution phase:**

   * just before flatpak execution, you can have shell script for handling anything you want with the set variables
**Example:** Importing Discord data from KDE to GNOME may seem broken until caches are cleared.
´´´
Scripting area:
rm -rf `"$DATA"`/cache
rm -rf `"$DATA"`/.cache
rm -rf `"$DATA"`/.ld.so
´´´

   * The runner runs `flatpak run $APPID`.
   * Option: provide a toggle to run with a terminal (off by default).





## 5. Practical Considerations & Edge Cases

* **Permissions:** Offloading system-installed Flatpaks often requires elevated privileges to manipulate `/var/lib/flatpak`. User-level Flatpaks in `~/.local/share/flatpak` are easier to manage without `sudo`.
* **Broken Symlinks:** The runner must safely detect and remove stale symlinks to avoid corrupting a user's real data.
* **Depth & Naming Constraints:** To keep discovery simple and reliable, the runner restricts search depth for AppIDs and requires certain naming patterns when placed inside subfolders.
* **Application-Specific Behavior:** Some apps store state in unexpected places (e.g., system caches, hardware-specific directories). The approach documents known exceptions and suggests workarounds.
* **Compatibility:** Not all Flatpaks or applications will behave identically across desktop environments; some manual adjustments may be necessary.
* **Security & Sandboxing:** Redirecting data via symlinks respects Flatpak's sandboxing model at the filesystem level, but any approach that changes data paths should be reviewed for sandbox and permission implications.
* **Safe defaults & explicit user action:** Because the mechanism touches user data paths, the runner defaults to `AppID=0` (no action) and an `IGNORE_LIST` to prevent accidental modifications. The user must explicitly set a real `AppID` to enable behavior.
* **Optional executable wrapper:** An optional executable binary wrapper can be provided to bypass execution flags for file managers that don't allow running shell scripts directly — mark this as RENAMABLE if desired.
> Note on offline installation can work with this method but it is kinda broken
* **Flatpak improvement:** While the symlink method can work great. It's not guaranteed that it will always work.
So, I'm trying to keep it separate. The ideal portable program should automatically adapt for your setup. So on the future if Flatpak adds an option for changing the location of those directories paths, this theoretically could also work.

One advantage of portable apps is offline installation. The current method relies mostly on downloading Flatpaks normally (online) and using symlinks for offloading. Offline Flatpak installation with just `"$BIN"`  files is complex and may be better with future Flatpak improvements, but this is not a core part of the approach today.
