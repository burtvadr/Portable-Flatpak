# Portable-Flatpak
Portable Flatpak logic that encapsulates app execution, user data, overrides, icons, and optional binaries inside a single movable folder. Copy, rename, and run Flatpaks anywhere using smart detection, and zero config files - Windows/MacOs like portability on Linux.

# Practical Pattern for Portable Flatpaks

This document describes a practical approach to achieving easier **portability** for applications on Linux — specifically with Flatpaks — by encapsulating the application, execution logic and persistent user data inside a single, movable directory. 

> **Goal:** transferring an application and its configuration between PCs should be as simple as copying a folder from a desktop to a laptop and running the program instantly.
---

## 1. Summary

The current concept of “portable software” in the Linux ecosystem is fragmented. Formats like AppImage offer single-file portability, but they typically do **not** encapsulate *user data* (configurations, saves, caches) inside the same container, which leads to data fragmentation.

This approach provides a pragmatic mechanism to encapsulate both application execution logic and persistent user data — and optionally binaries — into a single movable directory. The folder is driven by an embedded master logic that acts as middleware between the user and Flatpak. The launcher detects location and names and launches the application.

This is a tooling-first, practical example intended to work today rather than a formal standard, but it can inform future Flatpak improvements designed with portability in mind and flexibility in mind.
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

* **Adaptable:** Users should be able to rename and move most directories freely. The script should detect the folder's current absolute path and still work when the folder moves directories or names.
* **No metadata files:** configuration internally to avoid extra files and clutter.
* **Expressive layout:** Users should be free to organize the portable directory to their taste; a sensible default structure should exist.

This is a tooling-first approach intended to work now rather than a formal standard.

### 3.2 Possibles Directories Structures

You should be able to have a flexible structure that preserves your needs, with the freedom to name everything whatever you want

```
[Portable Application Folder]/      # renameable
├── runner                          # one-click run application (Entry Point) # renameable
├── stuff                           # (Optional) anything the user would want to keep; should never break anything # renameable
├── binaries                        # (Optional) Offloaded Flatpak binaries for ~/.local/share/flatpak/app/[AppID] # renameable
└── data                            # (On by default) Persistent User Data for ~/.var/app/[AppID]  # renameable
```

or, in a minimal layout:

```
[Portable Application Folder]/      # RENAMABLE
├── runner                          # one-click run application (Entry Point) # RENAMABLE
├── stuff                           # (Optional) anything the user would want to keep; should never break anything # RENAMABLE
└── [AppID]/                        # (On by default) Persistent User Data # RENAMABLE
```
if you want to go even more minimal
```
[Portable Application Folder]/      # RENAMABLE
├── Brave                           # AppID=com.brave.Browser one-click run application (Entry Point) # RENAMABLE
├── Discord                         # AppID=com.discordapp.Discord one-click run application (Entry Point) # RENAMABLE
├── Steam                           # AppID=com.valvesoftware.Steam one-click run application (Entry Point) # RENAMABLE
```
this effectively work as a script that just installs flatpak, adds flathub, installs the aplication by its [AppID] and run your program with one click

### 3.3 limitations

** Hard Constraints:**
* The user should never rename or move the folder while the app is running.

** soft Constraints:**
* [DATA] folders cannot be searched more than two directories deep (e.g., `[Portable Path]/[things]/[stuff]/[AppID]` is invalid by default).
* naturally avoid ntfs by checking location, sending a notify-send "Script Error" "NTFS unsupported, Move into a ext4 drive!" -u critical # (optional, no ntfs by default) 



### 3.3 Detection Mechanism 

* language assumptions, (on by default), `dat`´ `data` `dados` are detected as the [DATA] directory, same with bin/binaries/binarios will be detected as [BIN] , you should have control of the assumptions
* The system should detect what the user wants to keep automatically by the presence of directories so users generally don’t need to edit text files or use the terminal for typical workflows.

**user A — Data Persistence**
* **Trigger:** Presence of [DATA] and absence of [BIN]
* **results:** Keep only user saves/configuration portable; binaries remain on the host. Binaries are retrievable from Flathub.
**user A.2 — Binary Offloading**
* **Trigger:** Presence of [DATA] And presence [BIN]
* **results:** Store both binaries and user data on the portable directory. 
**user C — Click and run**
* **Trigger:** absence of [DATA] And absence of [BIN] 
* **results:** effectively just installs the application and executes it

if language assumptions are off, or the user changed the [DATA] directory for something new-
the detection of flatpaks [DATA] directory can be discovered by this this logic (examples preserved):

```
burtjo@Linux:/SSDpkg/packages/Program$ tree -a -L 2
.
├── перевод
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
└── runner
8 directories, 1 file
```
all flatpaks [DATA] directory have files like this inside, so you can enfer its a [DATA] directory by checking if there is more than 2 of this list of folders and hidden folders
(`cache`, `.cache`, `.config`, `config`, `data`, `.ld.so`, `.local`, `.nv`, `.pki`, `.var`)

```
burtjo@Linux:/SSDpkg/packages/Program$ tree -a -L 2
.
├── двоичный
│   ├── current -> x86_64/stable
│   └── x86_64
└── перевод
```
all flatpaks [BIN] directories have a (`x86_64`) folder and a and a symbolic link (`current`) -> (`x86_64/stable`)
so its easy to detect without language assumptions


```
burtjo@Linux:/SSDpkg/packages/Program$ tree -a -L 3
.
├── executables
│   ├── program stuff
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
└── runner

14 directories, 2 files
burtjo@Linux:/SSDpkg/packages/Program$ 
```
so even if a user has this crazy setup, the script should be able to know what is what

**language assumptions should help to install the application on a portable form, because the folders would be empty**
*  language assumptions help creating the necessary files for then you have the freedom for changing the names 

## 4. My Stretch: Desktop Integration

* (on by default) you should be able to disable desktop integration.
 in case you don't want desktop entries on your desktop enviroment ,so you can only run the application by clicking `runner`

* The system should detect what the user wants to keep automatically by the presence of directories so users generally don’t need to edit text files or use the terminal for typical workflows.
this also applies for custom desktop file and icons

```
[Portable Application Folder]/   
├── runner                      
├── stuff                    
├── icon.png        #should work with any name, any image format, with or without .png .svg. ico. .jpg .jpeg or .webp          
├── binaries               
└── data                
```

by dragging any image inside, automatically detect it at launch, apply the image on the desktop file for flatpaks by editing the icon= path on the system wide install at
 `/var/lib/flatpak/exports/share/applications/[AppID]` 
or for a --user install at
 `~/.local/share/flatpak/exports/share/applications/[AppID]`
 
```
[Portable Application Folder]/   
├── runner                         
├── stuff                    
├── icon.png        #should work with any name, any image format, with or without .png .svg. ico. .jpg .jpeg or .webp          
├── custom.desktop                 
├── binaries/                     
└── data/                          
```

or if a desktop file is already included on the example above, just edit this desktop file itself

### 5 Symlink Trick

Flatpak looks for user data in 
 `~/.var/app/[AppID]`
and user install binaries at
 `~/.local/share/flatpak/app/[AppID]`
or system wide at
 `/var/lib/flatpak/app/[AppID]`

* The Symlinks "tricks" Flatpak by redirecting that location to the portable folder — without patching Flatpak.

### 6 Practical execution logic

0. **back to basics:** ( on by default) Installs Flatpak and adds flathub. Auto detect your distro

1. **Launch phase:** The user executes `runner` #should be renameable 

2. **Detection phase:** 
   * the system computes its current absolute path (for example `/media/user/USB/Games/Osu`).
   * attempt to detect/check the [AppID],
   * mark as a valid Flathub [AppID] if it contains at least 3 dot-separated segments, regardless of what the segments contain.
   * prioritize if a [AppID] is on located on the `runner` name, if it renamed to [AppID] ( or [AppID].py or [AppID].exe or [AppID].sh depending on how this is implemented )
   * if a [AppID] is found conflicting with [AppID] found in directories names, just abort,
```
burtjo@Linux:/SSDpkg/packages/[AppID]$ tree -a -L 3
.
├── executables
│   ├── [AppID]
│   │   ├── current -> x86_64/stable
│   │   └── x86_64
│   └── sister draws
├── informations
│   ├── anything
│   └── [AppID]
│       ├── cache
│       ├── config
│       ├── data
│       ├── .ld.so
│       └── .local
└── [AppID]        # runner here, another [AppID] is cached internally, like AppID=[AppID] by default it should be AppID=0 (cache should be self contained, no extra files)

```
above are places where the user can have the [AppID] can be found
if `runner` is named as a [AppID], it should be at the top priority, `runner` with a [AppID] effectively overwrites internal AppID cache. 
  
   * [DATA] folders cannot be searched more than two directories deep (e.g., `[Portable Path]/[things]/[stuff]/[AppID]` is invalid by default).
   * flathub [AppID] confirmation check, instead of using the "at least 3 dot-separated segments, regardless of what the segments contain" method. to confirm the the found [AppID].
   just check if it is a valid id, by checking on flathub, (on by default) could be slower.
   * if icon exist, and store [ICON] as its path any image file, .png .svg. ico. .jpg .jpeg or .webp, with or without the extension, 
   * define [BIN_TARGET]. by user install `~/.local/share/flatpak/app/[AppID]`. by default, you should be able to configure if you want system wide installs `/var/lib/flatpak/app/[APPID]`
   * define if [LOCAL_DESKTOP_FILE] exist, it is defined at any plain text file with a header "[Desktop Entry]" 
   * define [DESKTOP_FILE]. by user install `~/.local/share/flatpak/exports/share/applications/[AppID]`. by default, you should be able to configure if you want system wide installs `/var/lib/flatpak/exports/share/applications/[AppID]` 
   * defines [OVERRIDES] (on by default) change overrides at `~/.local/share/flatpak/overrides/[AppID],` to cached inside the `runner` (no extra files), so its portable,

* **Automating permitions** (optional, on by default) will be added before launching the application the permission override at for `~/.local/share/flatpak/overrides/[AppID] `
* **Importing and exporting Permissions** 
the script should be able to at launch import or export the flatpaks override at ~/.local/share/flatpak/overrides/[AppID] via a launch command ./runner --export-overrides 
or --import-overrides, with a twist, it will change users names by $USER to accommodate new users name, in case you have a different user name on your new system. you can disable this behavior.

```
burtjo@Linux:~/.local/share/flatpak/overrides$ cat com.valvesoftware.Steam 
[Context]
devices=dri
filesystems=/media/burtjo/SSD/SteamLIB;/media/burtjo/SSDpkg/Gaming/Steam
```
gets converted to 
```
jason@Linux:~/.local/share/flatpak/overrides$ cat com.valvesoftware.Steam 
[Context]
devices=dri
filesystems=/media/jason/SSD/SteamLIB;/media/jason/SSDpkg/Gaming/Steam
burtjo@Linux:~/.local/share/flatpak/overrides$ 
```
/media gets converter to /run/media for fedora like distros, you can disable this behavior.
```
jason@Linux:~/.local/share/flatpak/overrides$ cat com.valvesoftware.Steam 
[Context]
devices=dri
filesystems=/run/media/jason/SSD/SteamLIB;/run/media/jason/SSDpkg/Gaming/Steam
burtjo@Linux:~/.local/share/flatpak/overrides$ 
```

   * ./runner --export-overrides # get the overrides from inside of your run script to your `~/.local/share/flatpak/overrides/[AppID]`
   * ./runner --import-overrides # saves the overrides at `~/.local/share/flatpak/overrides/[AppID]` on a section inside the shell script, the a "cache" the caches lives inside shell script, so no extra files or directories
   * those 2 only work after AppID= was marked with the app by the user or automatically by the script,
   * by default, always automatically import overrides if the `~/.local/share/flatpak/overrides/[AppID]` dont yet exist or is empty.
   * check if the application is already installed,
 
3. **if Its alredy installed:**

   * if it already is installed, compare how its installed with how its the `[Portable Path]` configuration. determined by the layout previously.
   * If `~/.var/app/[AppID]` is a broken symlink (from a previous location), remove it.
   * if `~/.var/app/[AppID]` its not a symlink and the user language assumption-
   * check if LANGAGE_INFERRING_[DATA] directories are empty
   * if they are, move the data from `~/.var/app/[AppID]/` to the LANGAGE_INFERRING_[DATA] directory,
   * be aware of `data/[AppID]` `dados/[AppID]` `dat/[AppID]` setups as well, 
   * and If `~/.var/app/[AppID]` is a standard directory, rename it by appending a rotation suffix (e.g., `com.vendor.org` → `com.vendor.org1`) so the original is backed up. If `com.vendor.org1` already exists, rotate to `com.vendor.org2`, and so on
   * If `[BIN_TARGET]` is a broken symlink (from a previous location), remove it.
   * If `[BIN_TARGET]` is a functional symlink, delete it to prepare for a new link.


4. **if its already installed at a oposing mode(sytem wide or user) other way around aware** 
   
   * if `/var/lib/flatpak/app/[AppID]` is a working directory or a working symlink,
   * wipe internal `[BIN]` and move `[BIN_TARGET]/x86_64` to `[BIN]/` then symlink `[BIN]/x86_64/stable` to `[BIN]/current` 
   * if its not using a [LOCAL_DESKTOP_FILE] and DESKTOP_INTEGRATION=1 
   * move `/var/lib/flatpak/app/[AppID]/current/active/export/share/applications/[AppID]` to `~/.local/share/flatpak/app/[AppID]/current/active/export/share/applications/[AppID]`
   * and symlink `~/.local/share/flatpak/app/[AppID]/current/active/export/share/applications/[AppID]` `~/.local/share/flatpak/exports/share/applications/[AppID]`
   * or if its using a [LOCAL_DESKTOP_FILE] or DESKTOP_INTEGRATION=0
   * proceeds with the symlink

5. **Linking** 
   * dinamically symlink what is needed based on the configuration of the directories layout
   * Symlink `[DATA]` to `~/.var/app/[AppID]`
   * Symlink `[BIN]` to `[BIN_TARGET]`
   * Symlink `[LOCAL_DESKTOP_FILE]` to `[DESKTOP_FILE]`

6. **if its not already installed** 
   * install according to configuration
   * installs the application normally, flatpak install --user [APPID] 
   * or system wide if was configured that way

7. **icon phase:**
   * if [ICON] exist
   * change the `[DESKTOP_FILE]` or `[LOCAL_DESKTOP_FILE]` (if it exists) `icon=` line to include the `[ICON]` absolute path


8. **Execution phase:**

   * just before flatpak execution, you can have shell script for handling anything you want with the set variables 
**Example:** Importing Discord data from KDE to GNOME may seem broken until caches are cleared.
# 2. Scripting area
´´´
rm -rf [DATA]/cache
rm -rf [DATA]/.cache
rm -rf [DATA]/.ld.so

´´´

   * The script runs `flatpak run [AppID]`.
   * Option: provide a toggle to run with a terminal (off by default).



´´´

# PORTABLE FLATPAK LAUNCHER 

# --- [ CONFIGURATION & DEFAULTS ] --------------------------------------------

# 1. AppID Configuration
MANUAL_APP_ID="0" 

# 2. Behavior Flags (0=Off, 1=On)
AUTO_INSTALL_FLATPAK=1      # Attempt to install Flatpak/Flathub if missing
FLATPAK_USER_MODE=1         # install flatpak in user mode
AUTO_INSTALL_APP=1          # Attempt to install the App if missing?
DESKTOP_INTEGRATION=1       # adds desktop entries on Desktop enviroment
USE_TERMINAL=0              # Launch app in a terminal window
AUTO_UPDATE_ICON=1          # Update .desktop icon if image found in folder
AUTO_PERMISSION_FIX=1       # Auto-grant access to this portable folder
IMPORT_OVERRIDES=1          # Auto-import stored permissions (overrides)
CONVERT_PATHS=1             # Convert /media <-> /run/media for portability
CUSTOM_DESKTOP_FILE=1       # auto detects desktop file and symlink it to 
LANGAGE_INFERRING=          # directories like data/dados/dat/ bin/binaries/binarios/binary will be check if they are the AppID
FLATHUB_CONFIRMATION=       # directories that are expected to be the app id will have the ID searched on flathub for the confirmation if they are a valid app id, if off, a simpler check for classic flatpak ip pattern will be used

# 2. Scripting area: 
´´´
rm -rf [DATA]/cache
rm -rf [DATA]/.cache
rm -rf [DATA]/.ld.so

´´´
# overrides storage area: 
# =============================================================================
´´´

´´´ 
# =============================================================================

LANGAGE_INFERRING_[BIN]=(bin,binary,binaries,binarios,Bin,Binary,Binaries,Binarios,BIN,BINARY,BINARIES,BINARIOS)
LANGAGE_INFERRING_[DATA]=(dat,data,dados,Dat,Data,Dados,DAT,DATA,DADOS)
´´´

## 5. Practical Considerations & Edge Cases

* **Permissions:** Offloading system-installed Flatpaks often requires elevated privileges to manipulate `/var/lib/flatpak`. User-level Flatpaks in `~/.local/share/flatpak` are easier to manage without `sudo`.
* **Broken Symlinks:** The script must safely detect and remove stale symlinks to avoid corrupting a user's real data.
* **Depth & Naming Constraints:** To keep discovery simple and reliable, the script restricts search depth for AppIDs and requires certain naming patterns when placed inside subfolders.
* **Application-Specific Behavior:** Some apps store state in unexpected places (e.g., system caches, hardware-specific directories). The approach documents known exceptions and suggests workarounds.
* **Compatibility:** Not all Flatpaks or applications will behave identically across desktop environments; some manual adjustments may be necessary.
* **Security & Sandboxing:** Redirecting data via symlinks respects Flatpak's sandboxing model at the filesystem level, but any approach that changes data paths should be reviewed for sandbox and permission implications.
* **Safe defaults & explicit user action:** Because the mechanism touches user data paths, the script defaults to `AppID=0` (no action) and an `IGNORE_LIST` to prevent accidental modifications. The user must explicitly set a real `AppID` to enable behavior.
* **Optional executable wrapper:** An optional executable binary wrapper can be provided to bypass execution flags for file managers that don't allow running shell scripts directly — mark this as RENAMABLE if desired.
> Note on offline installation can work with this method but it is kinda broken
* **Flatpak improvement:** While the symlink method can work great. It's not guaranteed that it will always work.
So, I'm trying to keep it separate. The ideal portable program should automatically adapt for your setup. So on the future if Flatpak adds an option for changing the location of those directories paths, this theoretically could also work.

One advantage of portable apps is offline installation. The current method relies mostly on downloading Flatpaks normally (online) and using symlinks for offloading. Offline Flatpak installation with just [BIN] files is complex and may be better with future Flatpak improvements, but this is not a core part of the approach today.

