# NBlood-Amiga / jfblood-3ds

A backport of [NBlood](https://github.com/nukeykt/NBlood) to JFBuild, ported to
the Nintendo 3DS. This repository is a fork of
[MrHuu/jfblood-3ds](https://github.com/MrHuu/jfblood-3ds).

---

## Building for 3DS (devkitPro)

### 1. Toolchain & dependencies

Install [devkitPro](https://devkitpro.org/wiki/Getting_Started) with the **3DS**
toolchain (`3ds-dev`), then install the runtime libraries the audio backend
needs. They are **not** part of the default `3ds-dev` group:

```sh
# inside the devkitPro MSYS2 / dkp-pacman shell
pacman -S --needed 3ds-libvorbisidec 3ds-libogg
```

> Without these you get `fatal error: tremor/ivorbisfile.h: No such file or directory`.

### 2. Clone with submodules

The engine and helper libraries live in git submodules
(`jfbuild`, `jfaudiolib`, `jfmact`). They are pinned to the
[`MrHuu/*`](https://github.com/MrHuu) repositories:

```sh
git clone --recurse-submodules https://github.com/Memorix101/jfblood-3ds.git
cd jfblood-3ds
# if you already cloned without --recurse-submodules:
git submodule update --init --recursive
```

### 3. Build

```sh
export DEVKITPRO=/opt/devkitpro
export DEVKITARM=$DEVKITPRO/devkitARM
make -f Makefile.ctr -j8
```

Output: `nblood.elf` (with symbols, for debugging) and `nblood.3dsx`.

The default target (`all`) builds both `nblood.3dsx` (with icon) and a `.cia`.
The `.3dsx` is what you run from the Homebrew Launcher.

#### `.cia` needs `bannertool` (optional)
Only the `.cia` target still needs `bannertool` (for the banner), which is no
longer shipped with devkitPro — so a plain `make -f Makefile.ctr` ends with
`bannertool: No such file or directory` **after** the `.3dsx` is already built.
That error is harmless if you only want the `.3dsx`. The icon (SMDH) is now
generated with `smdhtool` (ships with devkitPro), so the `.3dsx` builds with its
icon and **without** bannertool. If you do want the `.cia`, install a build of
[bannertool](https://github.com/Steveice10/bannertool) into
`$DEVKITPRO/tools/bin/`.

To (re)build just the icon'd `.3dsx` by hand:

```sh
smdhtool --create "NBlood" "NBlood port for 3DS" "MrHuu" rsrc/ctr/icon.png nblood.smdh
3dsxtool nblood.elf nblood.3dsx --smdh=nblood.smdh
```

---

## Installing the game on the SD card

The 3DS build changes its working directory to `sdmc:/3ds/NBlood` on startup
(see `jfbuild/src/ctrlayer.c`). Lay the SD card out like this:

```
sdmc:/
└── 3ds/
    ├── nblood.3dsx          <- launch this from the Homebrew Launcher
    └── NBlood/              <- create this folder; put the game data here
        ├── BLOOD.RFF
        ├── BLOOD.INI
        ├── SOUNDS.RFF
        ├── GUI.RFF
        ├── SURFACE.DAT
        ├── VOXEL.DAT
        ├── TILES000.ART ...
        ├── nblood.cfg          <- optional, see below
        └── (Cryptic Passage: CPART07.AR_ / CPART15.AR_, CRYPTIC.INI, ...)
```

- Copy the data files from your own copy of **Blood** (One Unit Whole Blood /
  Fresh Supply on GOG or Steam (DOS folder)) into `sdmc:/3ds/NBlood/`.
- The folder `sdmc:/3ds/NBlood/` must exist; the app does **not** create it.
- Filenames are case-sensitive on some setups (uppercase as above is safest).

### `nblood.cfg`

The config file `nblood.cfg` is **created automatically** in
`sdmc:/3ds/NBlood/` when you quit the game (it stores controls, video and audio
settings). You do **not** have to ship it. The copy in this repo is a
pre-tuned default (3DS button layout etc.); copying it to
`sdmc:/3ds/NBlood/nblood.cfg` is recommended so you start with sensible
controls instead of bare defaults, but it is optional.

---

## Notes for current toolchains

This project predates the GCC versions devkitPro now ships (GCC 14/15), which
turn several former warnings into hard errors. Two fixes are already applied in
this repo:

- `Makefile.ctr` adds `-fpermissive` to `CFLAGS` so the legacy C code
  (`-Wimplicit-function-declaration`, `-Wincompatible-pointer-types`,
  `-Wint-conversion`, `-Wreturn-mismatch`) compiles as it did originally.
- libctru gained `Result actInit(bool)`, which clashed with Blood's own
  `void actInit(bool)`. The libctru symbol is renamed locally where `<3ds.h>`
  is included (`blood.cpp`, `endgame.cpp`, `gamemenu.cpp`).
- The SMDH/icon rule in `Makefile.ctr` uses `smdhtool` instead of the missing
  `bannertool`.
- **Hang at "closing software" (HOME menu → Close).** Two issues in
  `jfbuild/src/ctrlayer.c`:
  1. The `APTHOOK_ONEXIT` hook called `backlightEnable()` (→ `gspLcdInit()` /
     `GSPLCD_PowerOnBacklight()`). That hook runs *inside* `aptMainLoop()` while
     the GSP right is being handed back to the system, where GSPLCD calls can
     deadlock; So the hook never returns and the close hangs. The GSPLCD call
     was removed from `ONEXIT`; it only sets the quit flag now.
  2. `main()` uses `gfxInit()` (not `gfxInitDefault()`), which does not register
     `gfxExit()`. Added `atexit(gfxExit)` + `return r;`, and `handleevents()` now
     checks `aptMainLoop()`'s return value and `exit(0)`s promptly on close so the
     applet is released cleanly.
