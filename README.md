# Portable-Flatpak
Portable Flatpak logic that encapsulates app execution, user data, overrides, icons, and optional binaries inside a single movable folder. Copy, rename, and run Flatpaks anywhere using a auto-detection, zero external config wrapper— Windows/Mac-like portability on Linux.

# Practical Pattern for Portable Flatpaks

This document describes a practical approach to achieving easier **portability** for applications on Linux — specifically with Flatpaks — by encapsulating the application, execution logic and persistent user data inside a single, movable directory. 

> **Goal:** transferring an application and its configuration between PCs should be as simple as copying a folder from a desktop to a laptop and running the program instantly.
---

## 1. Summary

The current concept of “portable software” in the Linux ecosystem is fragmented. Formats like AppImage offer single-file portability, but they typically do **not** encapsulate *user data* (configurations, saves, caches) inside the same container, which leads to data fragmentation.

This approach provides a pragmatic mechanism to encapsulate both application execution logic and persistent user data — and optionally binaries — into a single movable directory. The folder is driven by an embedded master logic that acts as middleware between the user and Flatpak. The launcher detects location and names and launches the application.

This is a tooling-first, practical example intended to work today rather than a formal standard, but it can inform future Flatpak improvements designed with portability and flexibility in mind 
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
[Portable Root]/                    # renameable
├── runner                          # one-click run application (Entry Point) # renameable
├── stuff                           # (Optional) anything the user would want to keep; should never break anything # renameable
├── binaries                        # (Optional) Offloaded Flatpak binaries for ~/.local/share/flatpak/app/[AppID] # renameable
└── data                            # (On by default) Persistent User Data for ~/.var/app/[AppID]  # renameable
```

or, in a minimal layout:

```
[Portable Root]/                    # RENAMABLE
├── runner                          # one-click run application (Entry Point) # RENAMABLE
├── stuff                           # (Optional) anything the user would want to keep; should never break anything # RENAMABLE
└── [AppID]/                        # (On by default) Persistent User Data # RENAMABLE
```
if you want to go even more minimal
```
[Portables Root]/                    # RENAMABLE
├── Brave                           # AppID=com.brave.Browser one-click run application (Entry Point) # RENAMABLE
├── Discord                         # AppID=com.discordapp.Discord one-click run application (Entry Point) # RENAMABLE
├── Steam                           # AppID=com.valvesoftware.Steam one-click run application (Entry Point) # RENAMABLE
```
this effectively work as a program that just installs flatpak, adds flathub, installs the application by its [AppID] and run your program with one click

### 3.3 limitations

**Hard Constraints:**
* The user should never rename or move the folder while the app is running.

**soft Constraints:**
* [DATA] folders cannot be searched more than two directories deep (e.g., `[Portable Path]/[things]/[stuff]/[AppID]` will not be searched by default).
* naturally avoid ntfs or others unsupported file systems, sending

```
├────────── [Portable Root]
├── cache
├── config
├── current -> x86_64/stable
├── data
├── .ld.so
├── .local
├── runner
└── x86_64

8 directories, 1 file
burtjo@Linux:/media/burtjo/SSDpkg/packages/Discord$ 

```
* this setup could also be a theoretically predictable pattern, but at least for me the complexity increases exponentially.
* so i wont include surgically individually detecting each directory on my implementation


```
notify-send "Error" "File system unsupported, Move into a linux File system drive\!" -u critical 
```
(optional, but no ntfs by default) 



### 3.3 Detection Mechanism 

**user A — Data Persistence**
* **Trigger:** Presence of [DATA] and absence of [BIN]
* **results:** Keep only user saves/configuration portable; binaries remain on the host. Binaries are retrievable from Flathub.

**user A.2 — Binary Offloading**
* **Trigger:** Presence of [DATA] And presence [BIN]
* **results:** Store both binaries and user data on the portable directory. 

**user C — Click and run**
* **Trigger:** absence of [DATA] And absence of [BIN] 
* **results:** effectively just installs the application and executes it

### 3.4 Declaring where is [DATA] and [BIN]  
```
├────────── [Portable Root]
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
└── runner                          # one-click run application (Entry Point) # renameable
8 directories, 1 file
```
all flatpaks [DATA] directories, at some level, have some of these files, so we can enfer [DATA] location by checking if there is more than 3 of this list of folders and hidden folders
(`cache`, `.cache`, `.config`, `config`, `data`, `.ld.so`, `.local`, `.nv`, `.pki`, `.var`)

```
burtjo@Linux:/SSDpkg/packages/Program$ tree -a -L 2
.
├── двоичный
│   ├── current -> x86_64/stable
│   └── x86_64
└── runner                          # one-click run application (Entry Point) # renameable
└── перевод
```
all flatpaks [BIN] directories have a (`x86_64`) folder and a and a symbolic link (`current`) -> (`x86_64/stable`)
so its easy a easy check for those two.

if [DATA] and [BIN] coulnt be determined using the method above we proceds with:**

### 3.5 language assumptions 

*  they help you to install the application, and the system know what are the [DATA] and [BIN] locations.
*  language assumptions help creating the necessary files for detection though the flatpak installation.
*  after the files are installed, you have the freedom for changing and moving the names, [DATA] and [BIN] can now be determined by its contents normally.
*  language assumptions are on by default. `dat`´ `data` `dados` are detected as the [DATA], `bin` `binaries` `binarios` as [BIN], # you should be free to control assumptions, by creating your own list or disabling completely

## 3.6 contents aware system. No configuration,

*   The system should detect with no extra configuration needed, what the user wants to keep portable. through the presence or absence of directories or files, so users don’t need to edit config files or use the terminal.

```
├────────── [Portable Root]
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
└── runner                 # one-click run application (Entry Point) # renameable
```
even this crazy setup, the system should be able to detect `eerm=[DATA]` and `program stuff=[BIN]`


## 4. My Stretch: Desktop Integration

* naturally, installing flatpaks adds a desktop entry on your Desktop environment, we want to preserve this behavior by default. #you should be able to disable desktop integration. so you can only run the application by clicking the `runner`

*automatically applies a custom icon, by detecting by the presence of any images inside the `[Portable Root]` so you don’t have to edit anything manually

```
├────────── [Portable Root]
├── runner                      
├── stuff                    
├── icon        #lets supposed its a svg, but is has no extension, should work without any problem, **never depend exclusively file extension, .png .svg. ico. .jpg .jpeg or .webp 
├── binaries               
└── data                
```

by dragging any image on `[Portable Root]` we should automatically apply by editing the `icon=` line on the desktop file, specifically for flatpaks:

system wide
 `/var/lib/flatpak/exports/share/applications/[AppID]` 
or for a --user install at
 `~/.local/share/flatpak/exports/share/applications/[AppID]`


 * users should also want to keep custom .desktop file and like everything here we should automatically by its presence, if the user don't want a desktop, just don't have it on [Portable Root]
```
├────────── [Portable Root]
├── runner                         
├── stuff                    
├── icon.png       #lets say its a png, should work with any iamge format, never depend exclusively on detecting the file extension, .png .svg. ico. .jpg .jpeg or .webp   
├── custom.desktop                 
├── binaries/                     
└── data/                          
```
and of course, for icons, if a desktop file is already included on the [Portable Root] just edit this desktop file itself, no need to touch  `~/.local/share/flatpak/exports/share/applications/[AppID]` or  `/var/lib/flatpak/exports/share/applications/[AppID]` 


### 5 Symlink Trick

 * Flatpak looks for user data in 
 `~/.var/app/[AppID]`

 
 *and user install binaries at
 `~/.local/share/flatpak/app/[AppID]`
  or system wide at
 `/var/lib/flatpak/app/[AppID]`

* Using Symlinks to achieve Portable applications, Specifically Flatpaks by redirecting that location to the portable folder — without patching Flatpak.

### 6 Practical execution logic

0. **back to basics:** automatically Installs Flatpak and adds flathub to your distro

1. **Launch phase:** The user executes `runner` #should be renameable 

2. **the system computes its current absolute path (for [Portable Root] example `/media/user/USB/Games/Osu`:** 

3. **the system checks the aplication [AppID]:**
   * if the [BIN] was successfully check if $BIN/current/active/export/bin/[AppID] exist, this location always has a file with the [AppID] 
   * but if it dont exist. there are others ways to check for a [AppID] bellow.
   * valid [AppID]'s is a constant and unique identifier of the application consisting of the reverse-DNS format {tld}.{vendor}.{product}.
```
├────────── [Portable Root]
│   ├── двоичный
│   │   ├── current -> x86_64/stable
│   │   └── x86_64
│   └── sister draws
├── informations
│   ├── anything
│   └── перевод
│       ├── cache
│       ├── config
│       ├── data
│       ├── .ld.so
│       └── .local
└── runner       # runner here
```
since the user is free to name and move files, he could just not include the [AppID] anywhere
for correctly determening the [AppID] we will store a value inside the runner
```
├────────── [Portable Root]
│   ├── двоичный
│   │   ├── current -> x86_64/stable
│   │   └── x86_64
│   └── sister draws
├── informations
│   ├── anything
│   └── перевод
│       ├── cache
│       ├── config
│       ├── data
│       ├── .ld.so
│       └── .local
└── runner       # runner here, AppID=0, by default (cache should be self contained, no extra files)
```
but still on this case, we know `[DATA]` and `[BIN]`, but the [AppID] is stored as AppID=0 so the system cannot predict, so it will just abort
   * if a [AppID] is found conflicting with [AppID] found in directories names, just abort,

   ```
├────────── [Portable Root]
│   ├── двоичный
│   │   ├── current -> x86_64/stable
│   │   └── x86_64
│   └── sister draws
├── informations
│   ├── anything
│   └── перевод
│       ├── cache
│       ├── config
│       ├── data
│       ├── .ld.so
│       └── .local
└── [AppID]       # runner here, AppID=0, by default (cache should be self contained, no extra files)
```
The user now change the `runner` name for a [AppID], if the internal [AppID] equals to AppID=0 we will store the [AppID] internally: 
   ```
├────────── [Portable Root]
│   ├── двоичный
│   │   ├── current -> x86_64/stable
│   │   └── x86_64
│   └── sister draws
├── informations
│   ├── anything
│   └── перевод
│       ├── cache
│       ├── config
│       ├── data
│       ├── .ld.so
│       └── .local
└── [AppID]       # runner here, AppID=[AppID], by default (cache should be self contained, no extra files)
```
Now the [AppID] is stored internally, the user is not free to rename the `runner` to anything he wants, for example:
   ```
├────────── [Portable Root]
├── executables
│   ├── двоичный
│   │   ├── current -> x86_64/stable
│   │   └── x86_64
│   └── sister draws
├── informations
│   ├── anything
│   └── перевод
│       ├── cache
│       ├── config
│       ├── data
│       ├── .ld.so
│       └── .local
└── execute       # runner here, AppID=[AppID], by default (cache should be self contained, no extra files)
```
app id can be determined by the internal `AppID=[AppID]` 
   ```
├────────── [Portable Root]
├── executables
│   ├── [AppID]
│   │   ├── current -> x86_64/stable
│   │   └── x86_64
│   └── sister draws
├── [AppID]
│   ├── anything
│   └── [AppID]
│       ├── cache
│       ├── config
│       ├── data
│       ├── .ld.so
│       └── .local
└── [AppID]       # runner here, AppID=0, by default (cache should be self contained, no extra files)
```
here are example places we check for possible [AppID] 
   * [AppID], [DATA] and [BIN] folders cannot be searched more than two directories deep (e.g., `[Portable Path]/[things]/[stuff]/[AppID]` is invalid by default).
   * override if internal cached AppID=0 if [AppID] is on located on the `runner` name. if the internal AppID= a non matching [AppID] compared to the  `runner` name, launchs a pop up, is you aplication [AppID_X] OR [AppID_Y]?
   * aware of examples like org.telegram.desktop, .desktop is not a extension, its part of the [AppID],
   * aware of examples like `org.telegram.desktop.sh` or `org.telegram.desktop.py`, or even just `org.telegram.desktop` without any extension.
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
4. **Definitions:**
   * Defines the $APPID for the succesully detected [AppID]
   * Defines its own absoulte path as variable for $PORTABLE_ROOT
   * Define [USER_MODE] for user install (default). or [SYSTEM_MODE] for system wide install
   * Defines `$BIN_FLATPAK` variable if choosen user mode or system wide:
   
     user mode at
     `$BIN_FLATPAK` = `~/.local/share/flatpak`. or for system wide `$BIN_FLATPAK` = `/var/lib/flatpak`.
     or for system wide at:
     
   * defines [BIN_TARGET] for `$BIN_FLATPAK/app/[AppID]`
   * define if [DESKTOP_FILE] exist at [Portable Root] , can be defined at any plain at the text file with a header "[Desktop Entry]" 
   * if a desktop file is present on the [Portable Root] define $PORTABLE_ROOT/[DESKTOP_FILE] For $DESKTOP_FILE if a desktop file is present on the [Portable Root] 
   * if no [DESKTOP_FILE] is present on the [Portable Root] 
   * define $DESKTOP_FILE AS `$BIN_FLATPAK/exports/share/applications/$APPID`.

   
   * defines user [OVERRIDES] (on by default) change overrides at `~/.local/share/flatpak/overrides/[AppID],` to cached inside the `runner` (no extra files), so its portable,

* **Automating permissions** (optional, on by default) will be added before launching the application the permission override at for `~/.local/share/flatpak/overrides/[AppID] `
* **Importing and exporting Permissions** 
the system should be able to at launch import or export the flatpaks override at ~/.local/share/flatpak/overrides/[AppID] via a launch command ./runner --export-overrides 
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

   * ./runner --export-overrides # get the overrides from inside of your runner to your `~/.local/share/flatpak/overrides/[AppID]`
   * ./runner --import-overrides # saves the overrides at `~/.local/share/flatpak/overrides/[AppID]` on a section inside the runner, the a "cache" the caches lives inside runner, so no extra files or directories
   * those 2 only work after AppID= was marked with the app by the user or automatically by the runner,
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
´´´
Scripting area:
rm -rf [DATA]/cache
rm -rf [DATA]/.cache
rm -rf [DATA]/.ld.so
´´´

   * The runner runs `flatpak run [AppID]`.
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

One advantage of portable apps is offline installation. The current method relies mostly on downloading Flatpaks normally (online) and using symlinks for offloading. Offline Flatpak installation with just [BIN] files is complex and may be better with future Flatpak improvements, but this is not a core part of the approach today.

