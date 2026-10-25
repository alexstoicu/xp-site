# Codename Eagle (1999) on Windows 11 — Reproducible Install Guide

A complete write-up of making *Codename Eagle* (Refraction Games / TalonSoft / Take-Two,
1999) install and run on a modern 64-bit Windows 11 PC, starting from a CloneCD disc
image. Written so a person — or an AI agent — can reproduce the whole thing without
repeating the trial and error this session went through.

This document contains **no game assets, no copyrighted files, and no cracked
executables**. The reader must supply their own legally owned copy of the game (a
CloneCD image: `.img`/`.ccd`/`.cue`/`.sub`, or an equivalent dump of the original disc).

---

## 1. Goal and final result

**Goal:** Take a 1999 retail CD image of Codename Eagle and produce a game that:
- Installs on 64-bit Windows 11 without any SafeDisc/DRM blocking it
- Runs without the original CD/disc image present at runtime
- Renders through a modern GPU (via a Direct3D/DirectDraw translation layer)
- Has its CD-audio soundtrack available as files
- Is officially patched to the last version that doesn't break singleplayer
- Can be redistributed as a single installer file that does all of the above

**Final result achieved:**
- A single self-contained installer (`CodenameEagle_Setup.exe`, built with Inno
  Setup, ~350 MB) that extracts the game, sets the one required registry key, and
  creates shortcuts — no ISO, no virtual CD-ROM drive, no manual steps.
- The game launches and runs full-screen through dgVoodoo2 on an NVIDIA GPU.
- Singleplayer works (patched to the 1.33 + singleplayer-hotfix combination).
- All 13 CD-audio tracks are ripped to `.ogg` files; background music playback is
  wired into the launcher but **disabled by default** (the game's own sound
  effects/voice lines were judged sufficient; music can be re-enabled by uncommenting
  two lines in the launcher `.bat`, or by re-running the installer with the "Enable
  looping background music" task checked).

---

## 2. Environment

| Item | Value |
|---|---|
| OS | Windows 11 Home 64-bit, build 26200 |
| CPU | Intel laptop CPU (11th gen, 12 logical processors) |
| GPU | Hybrid/Optimus laptop: Intel UHD Graphics (primary display adapter) + NVIDIA GeForce RTX 3050 Ti Laptop GPU (driver 595.71 at time of writing) |
| Display | 1920×1080 @ 144 Hz |
| Game source | CloneCD image: `<name>.img` (data track + 13 CD-audio tracks), `.ccd`, `.cue`, `.sub` |

Tools used (all freely available, no paid software):

| Tool | Version | Purpose |
|---|---|---|
| 7-Zip | 22.01 x64 | Reading the ISO9660 filesystem once de-raw'd |
| Python | 3.14 | Raw sector stripping, audio extraction, PE parsing, disassembly |
| `pefile` (Python package) | latest via pip | Parsing the game executable's PE headers/sections/imports |
| `capstone` (Python package) | 5.0.9 | x86 disassembly |
| `unshield` | 1.6.2 (Scoop community binary bucket) | Extracting the legacy InstallShield "ISc(" cabinet format — 7-Zip cannot read this format |
| ffmpeg | 8.1.2 | (available but not ultimately used for audio extraction — see §5) |
| dgVoodoo2 | 2.87.5 | DirectDraw/Direct3D-to-modern-D3D11/12 translation wrapper |
| Sysinternals Process Monitor (Procmon) | current (Oct 2026 download) | Tracing the game's file/registry access to diagnose the launch failure |
| Inno Setup | 6.7.3 | Building the final distributable installer |
| VLC | 3.0.16 | Playing the ripped `.ogg` music files (optional, launcher-invoked) |

No paid tools, no debuggers beyond free disassembly (Procmon + capstone), no
network services beyond fetching the above public tools and two official/community
patch files.

---

## 3. Root causes (every reason it failed, and how each was found)

### 3.1 The retail installer is SafeDisc-protected (identified, not bypassed)

The disc's root contained the classic SafeDisc v1.x fingerprint:

- `00000001.TMP` — SafeDisc marker file
- `clcd16.dll` / `clcd32.dll` — SafeDisc CD-authentication DLLs
- `dplayerx.dll` — SafeDisc loader (replaces a DirectPlay DLL)
- `drvmgt.dll` — installs/manages the SafeDisc kernel driver
- `secdrv.sys` — the SafeDisc kernel-mode driver itself

**Diagnosis method:** simply listing the extracted disc contents and recognizing
these well-documented SafeDisc filenames.

**Why it matters:** `secdrv.sys` is a 32-bit-only legacy kernel driver. It cannot
load on 64-bit Windows at all (no x64 build exists), and Microsoft additionally
shipped a security update years ago that disables/removes it outright. Running the
original `SETUP.EXE` or the SafeDisc-wrapped main game executable (`Game.exe`,
confirmed below) was never going to work on Windows 11 x64, regardless of any
registry or compatibility trick.

**Resolution path taken:** per explicit user instruction, this DRM was **not**
cracked or bypassed. Instead, the official 1.33 patch was found to ship a second,
*unprotected* copy of the main executable (see §3.3) — a legitimate way around the
problem that required no modification of any protected binary.

### 3.2 The retail installer's own front-end cannot run on 64-bit Windows at all

Later in the process, an attempt was made to run the *patch's* `SETUP.EXE` (which
is **not** SafeDisc-protected) to see if it would correctly finish registering the
game. It failed with Windows reporting it as a 16-bit application that cannot run.

**Diagnosis:** `file SETUP.EXE` reported `MS-DOS executable, NE version 5 for MS
Windows 3.10 (EXE) (GUI)` — a 16-bit "New Executable" stub. 64-bit Windows removed
NTVDM (the 16-bit subsystem) entirely; there is no compatibility-mode workaround
for this on 64-bit Windows. This is a hard, unconditional incompatibility.

**Resolution:** the patch's actual payload (`data1.cab`, in the legacy
InstallShield "ISc(" cabinet format) was extracted directly with `unshield`,
bypassing the 16-bit stub entirely. No functional loss results from this — the
stub's only job is to drive the 32-bit install engine, which was replaced by
manual extraction.

### 3.3 Two executables, two very different protection states

The 1.33 patch's cabinet contains (among many other files) three top-level
executables: `GAME.EXE`, `EURO.EXE`, and `ce.exe`.

**Diagnosis:** comparing PE structure and searching for known SafeDisc strings in
each:

- `GAME.EXE` (which, on a case-insensitive Windows filesystem, overwrites the
  original `Game.exe`): 8 PE sections, contains the `BoG_` SafeDisc marker string,
  and imports/references `secdrv`/`drvmgt` — **still fully SafeDisc-wrapped.**
  Actually launching it produces a Windows dialog: *"Access Denied — Please login
  with administrator privileges and try again"* (SafeDisc's driver-install attempt
  failing).
- `ce.exe`: only 3 PE sections, **zero** SafeDisc strings/imports. This is a
  legitimate, publisher-shipped unprotected build — not something cracked by this
  session. `ce.exe` became the executable the final launcher uses.

### 3.4 `ce.exe` still refused to start: "Codename: Eagle not installed"

Running `ce.exe` directly produced a Windows message box, title "Codename Eagle",
text **"Codename: Eagle not installed"**, and the process then exited. This
happened consistently regardless of working directory or elevation.

**Diagnosis process (in order):**

1. **Process Monitor tracing** of `ce.exe`'s full registry/file activity showed it
   reading `HKLM\SOFTWARE\WOW6432Node\Take2\Codename Eagle\SrcDisk\Status` →
   `NAME NOT FOUND`. (Note: this is the 32-bit WOW6432Node view, because `ce.exe`
   is a 32-bit process; on 64-bit Windows this redirection is automatic and
   transparent.)
2. Setting a `Status` value (first attempt: `REG_SZ "1"`) got further in the trace
   but produced the *identical* dialog — proving the string was reused generically
   for more than one failure path, and that black-box guessing of registry values
   was not going to finish the job.
3. Web research turned up a 2008-era community blog documenting that v1.3.6
   required two **separate** registry values under the same key: `Drive` (the CD
   drive letter) and `Path` (the install directory) — not just one. Both were set.
   `Drive`'s value was pointed at a disc image mounted with Windows' native
   `Mount-DiskImage` (which presents as a real `DRIVE_CDROM` type device). This
   *still* produced the identical dialog.
4. With explicit user permission, the executable was disassembled (Python +
   `pefile` + `capstone`) to find the actual check. The string's one cross-reference
   led to this logic (addresses are from this specific build; do not assume they
   generalize to other builds/versions):

   ```
   mov ecx, 0xEC62D         ; 0xEC62D = 968237 decimal
   call sub_481D00
   test eax, eax
   jne  <continue normal startup>
   ... else: MessageBoxA(NULL, "Codename: Eagle not installed", "Codename Eagle", ...)
   ```

   `sub_481D00` does, exactly:
   ```
   RegOpenKeyExA(HKEY_LOCAL_MACHINE, "Software\Take2\Codename Eagle\SrcDisk",
                 0, KEY_READ, &hKey)
   RegQueryValueExA(hKey, "Status", NULL, &type, &data, &size)
   RegDeleteValueA(hKey, "Status")      ; present in the disassembly; empirically
                                         ; this call did not appear to execute in
                                         ; Procmon traces across repeated test runs —
                                         ; unverified why, but the net effect observed
                                         ; was that Status persisted and did NOT need
                                         ; to be re-written before every launch
   RegCloseKey(hKey)
   return (data == 0xEC62D) ? TRUE : FALSE
   ```

5. **The actual bug:** the first disassembly-informed fix used the wrong *type*
   (`REG_SZ` instead of `REG_DWORD`) **and** the wrong *value* — a hand arithmetic
   slip converting `0xEC62D` to decimal (966189, instead of the correct 968237).
   Fixing both (type **and** exact value) resolved the dialog completely.

**Root cause, stated plainly:** `ce.exe` expects a one-time installation marker —
a `REG_DWORD` named `Status` under
`HKLM\SOFTWARE\Take2\Codename Eagle\SrcDisk` equal to **968237 (0xEC62D)** — that
the original 1999/2000-era installer would have written. Manually extracting the
game's files (to work around the SafeDisc and 16-bit-installer problems above)
skipped the step that wrote this value, so it had to be set by hand.

### 3.5 The `Drive` value and the mounted virtual CD-ROM were a dead end, not a real requirement

After the real fix (3.4) was found, the virtual CD-ROM mount was removed entirely
and `ce.exe` was re-tested. It launched exactly as well without it — it progressed
*further* into actual Direct3D device creation than it had with the drive mounted.

**Conclusion:** the disassembled check never inspects `Drive` or any physical/
virtual CD-ROM drive at all. The `Drive` value's *content* appears to be
unvalidated — it only needs to exist as a string under the same key, for reasons
not identified in this session (`unverified`: possibly read by a feature never
exercised, such as playing intro movies from physical media — the FMVs are already
installed as local files in this setup, so this was never tested).

### 3.6 Admin rights were not actually required for `ce.exe`

A community source claimed admin rights were required to run the (older, patched)
game. This was tested directly: `ce.exe`, once the registry fix in §3.4 was in
place, launched identically whether started elevated or not. **Unverified** whether
this differs for other versions/patches; for the 1.33 + hotfix + `ce.exe`
combination used here, no elevation is needed at runtime (only the *installer*,
because it writes to `HKLM`, requests admin).

---

## 4. Exact step-by-step process

Placeholders used below: `<GAME_ROOT>` = the top-level working folder (this
session used `C:\Games\Codename Eagle`), `<GAME_DIR>` = the final install folder
inside it (this session used `<GAME_ROOT>\Game`), `<DISC_IMG_DIR>` = wherever your
CloneCD dump lives.

### 4.1 Inventory

1. Identify GPU/driver/OS (`nvidia-smi`, `dxdiag /t`, or Settings → System → About).
2. Confirm 7-Zip is installed (`C:\Program Files\7-Zip\7z.exe`); if not, install it.
3. Confirm a modern Python 3 is available for the scripting steps below.

### 4.2 De-raw the data track from the CloneCD image

CloneCD `.img` files store raw `2352`-byte CD sectors (sync + header + 2048 bytes
of user data + EDC/ECC), not a plain ISO9660 filesystem. 7-Zip cannot open this
directly. The data track's sector range must be computed from the `.cue` file.

1. Open the `.cue` file and note the data track's `INDEX 1` position (always
   `00:00:00` for track 1) and the next track's `INDEX 1` position — this is the
   data track's end, in `MM:SS:FF` (minutes:seconds:frames, 75 frames/second).
2. Convert `MM:SS:FF` → sector count: `sectors = (MM*60 + SS)*75 + FF`.
3. Strip each `2352`-byte sector down to its `2048` bytes of user data (offset
   `16`, length `2048`) for every sector from `0` up to (but excluding) the next
   track's start sector. Write the result out as a plain `.iso`.

   ```python
   SECTOR, USER_OFFSET, USER_SIZE = 2352, 16, 2048
   with open("<DISC_IMG_DIR>\\disc.img", "rb") as f, open("track1.iso", "wb") as out:
       for i in range(NUM_SECTORS):          # NUM_SECTORS computed as above
           f.seek(i * SECTOR)
           out.write(f.read(SECTOR)[USER_OFFSET:USER_OFFSET + USER_SIZE])
   ```
4. Sanity check: `7z l track1.iso` should now list a normal ISO9660 file tree. You
   can cross-check the `.ccd` file's `[Entry N]` records (`TrackNo`/`PLBA` fields)
   against your sector-count math — they should agree exactly.

### 4.3 Identify the installer and copy protection

1. `7z l track1.iso` and look for `SETUP.EXE`, `setup.ins`, `_INST32I.EX_`,
   `data1.cab`, `_sys1.cab`, `_user1.cab` → classic **InstallShield** project.
2. Look for `00000001.TMP`, `clcd16.dll`, `clcd32.dll`, `dplayerx.dll`,
   `drvmgt.dll`, `secdrv.sys` → **SafeDisc** (see §3.1). **Do not attempt to
   install/run through this path on 64-bit Windows; it will not work and should
   not be bypassed without the game owner's explicit, informed decision.**

### 4.4 Extract the actual game files without running the protected installer

`data1.cab` is **not** a Microsoft CAB file — it uses the older InstallShield
"cabinet" format (magic bytes `ISc(`), version 5/6. 7-Zip reports
`Cannot open the file as archive` for this format.

1. Get `unshield` (e.g. the Scoop community binary bucket ships a prebuilt
   Windows x64 7z archive; verify its SHA-256 against the published manifest
   before trusting it).
2. Pull `data1.cab` out of the ISO (`7z x track1.iso data1.cab`).
3. `unshield.exe l data1.cab` to confirm it lists full relative paths (e.g.
   `Program Executable Files\CE\Game.exe`).
4. `unshield.exe -d <GAME_DIR> x data1.cab` to extract everything.
5. Flatten the resulting `Program Executable Files\CE\...` nesting up to
   `<GAME_DIR>` directly.
6. Separately extract the `CUTSCN\` folder (cinematics) from the ISO root — it is
   **not** inside `data1.cab`:
   `7z x track1.iso -o<GAME_DIR> CUTSCN`

### 4.5 Rip the CD-audio soundtrack

The music tracks are plain 44.1 kHz/16-bit/stereo Redbook CD audio, stored
back-to-back in the same `.img` after the data track. Using the same `.cue`
timestamps:

1. For each audio track, convert its `MM:SS:FF` start (and the next track's start,
   or end-of-file for the last track) to a byte range: `sector * 2352`.
2. Read that exact byte range directly out of the `.img` and write it as a `.wav`
   (2 channels, 16-bit, 44100 Hz) using Python's `wave` module — this avoids any
   floating-point rounding that `ffmpeg -ss`/`-t` seeking on a raw stream can
   introduce.
3. Convert the `.wav` files to `.ogg` (e.g. `ffmpeg -i trackNN.wav -c:a libvorbis
   -q:a 5 trackNN.ogg`) and discard the `.wav` files afterward.
4. Cross-check: total `.img` size ÷ 2352 should exactly equal the lead-out sector
   number recorded in the `.ccd` file's last TOC entry. If it doesn't divide
   evenly, your track math is off somewhere.

### 4.6 Apply the official patch (and only that patch, for singleplayer)

**Do not patch past the version documented to still support singleplayer.**
Community documentation is consistent that patches after 1.33/1.36 break
singleplayer and this was never fixed upstream.

1. Download the official "1.33 Cumulative" patch and the separate "1.33
   Singleplayer Hotfix" from a trustworthy archive (this session used
   PCGamingWiki's community file host; file-hosting sites sometimes require
   following a timed "confirm download" redirect rather than a direct link —
   inspect the HTML response for a `confirm=1&...csrfKey=...` link if a direct
   download returns an HTML page instead of the file).
2. The 1.33 cumulative patch is itself an InstallShield package — extract its
   `data1.cab` with `unshield` exactly as in §4.4 (its 16-bit `SETUP.EXE` stub
   cannot run on 64-bit Windows at all — see §3.2 — so don't try to run it).
3. Copy the extracted patch's `Program Executable Files\*` tree over
   `<GAME_DIR>`, merging/overwriting. On a case-insensitive filesystem, files
   like `GAME.EXE` correctly overwrite `Game.exe` automatically.
4. **Apply the singleplayer hotfix *after* the cumulative patch, not before** —
   the cumulative patch's own `level1\data1.bin` is the one the hotfix fixes.
   The hotfix in this session was a single raw file (`data1.bin`) meant to
   directly replace `<GAME_DIR>\level1\data1.bin` (same file size as the one
   already there — compare sizes/hashes to confirm you're replacing the right
   file, not extracting an archive).
5. Use `<GAME_DIR>\ce.exe` (see §3.3) as the executable going forward, never the
   `Game.exe` that the patch also drops in.

### 4.7 Fix the registry so `ce.exe` will actually launch

This is the fix found in §3.4. On the machine that will run the game:

```
reg add "HKLM\SOFTWARE\Take2\Codename Eagle\SrcDisk" /v Path /t REG_SZ /d "<GAME_DIR>" /f
reg add "HKLM\SOFTWARE\Take2\Codename Eagle\SrcDisk" /v Status /t REG_DWORD /d 968237 /f
reg add "HKLM\SOFTWARE\Take2\Codename Eagle\SrcDisk" /v Drive /t REG_SZ /d "C:" /f
```

Notes:
- Run this `reg add` from an **elevated** prompt — `HKLM` is not writable by a
  standard user token.
- Run it as a plain 32-bit `reg.exe` invocation (the normal one on 64-bit
  Windows is fine) so WOW64 redirection puts it in
  `HKLM\SOFTWARE\WOW6432Node\Take2\Codename Eagle\SrcDisk`, where the 32-bit
  `ce.exe` will actually look for it. If you ever read this key back to verify,
  remember to read the `WOW6432Node` path, not the bare `SOFTWARE` path.
- `Status` **must** be `REG_DWORD` with the exact decimal value `968237`
  (= `0xEC62D`). A string value, or any other number, will not satisfy the check.
- `Drive`'s content does not appear to be validated (§3.5) — any string is fine.
- In testing this value persisted across repeated launches without needing to be
  re-written (§3.4, step 4's `RegDeleteValueA` did not appear to actually fire in
  practice) — treat this as a one-time setup step, not something the launcher
  needs to redo every run.

### 4.8 Install dgVoodoo2 for modern-GPU rendering

1. Download dgVoodoo2 (this session used v2.87.5) from its GitHub releases.
2. Copy the 32-bit DLLs into `<GAME_DIR>` (next to `ce.exe`): `DDraw.dll`,
   `D3DImm.dll`, `D3D8.dll`, `D3D9.dll` (the game uses DirectDraw/old Direct3D;
   dropping in all four is harmless and covers whichever the engine actually
   loads at runtime).
3. Copy `dgVoodooCpl.exe` (the config GUI) into `<GAME_DIR>` too.
4. Create/edit `<GAME_DIR>\dgVoodoo.conf` (a plain text file, heavily commented by
   dgVoodoo2 itself). Settings used in this session:
   - `[General] ScalingMode = stretched_ar` (fill the screen, keep aspect ratio)
   - `[DirectX] Resolution = max_fhd` (force up to 1920×1080 output)
   - `[DirectX] Filtering = bilinear`
   - `[DirectX] ForceVerticalSync = true`
   - Everything else left at dgVoodoo2's shipped defaults.
5. The game can also be reconfigured any time afterward by running
   `dgVoodooCpl.exe` directly (GUI).

### 4.9 Build a launcher

A simple batch file covers starting the game (and, optionally, looping the ripped
music in the background via VLC):

```bat
@echo off
setlocal
set GAMEDIR=%~dp0Game
set VLC="C:\Program Files\VideoLAN\VLC\vlc.exe"
cd /d "%GAMEDIR%"

rem Background music is OFF by default. To re-enable, uncomment:
rem if exist %VLC% (
rem     start "CE Music" /min %VLC% --intf dummy --loop --no-video -q "%GAMEDIR%\MUSIC"
rem )

start /wait "" "%GAMEDIR%\ce.exe"
taskkill /f /im vlc.exe >nul 2>&1
endlocal
```

### 4.10 Package it all into one installer

An Inno Setup script (`.iss`) that:
- Packages the entire finished `<GAME_DIR>` tree (`[Files] Source:
  "<GAME_DIR>\*"; DestDir: "{app}"; Flags: recursesubdirs createallsubdirs`)
- Writes the three registry values from §4.7 under
  `Root: HKLM; Subkey: "SOFTWARE\Take2\Codename Eagle\SrcDisk"` — **do not**
  mark the installer `ArchitecturesInstallIn64BitMode`; leaving the installer as
  a plain 32-bit process makes its own `HKLM\SOFTWARE` writes auto-redirect to
  `WOW6432Node`, matching what the 32-bit game reads
- Creates Start Menu / optional desktop shortcuts to `{app}\ce.exe` and
  `{app}\dgVoodooCpl.exe`
- Has a `[Tasks]` checkbox (unchecked by default) to enable background music,
  and writes the launcher `.bat` accordingly via `[Code]` (`SaveStringsToFile`)
- Offers to launch the game immediately after install (`[Run] ... Flags: nowait
  postinstall skipifsilent`)
- Sets `PrivilegesRequired=admin` (needed to write `HKLM`)

Compile with `ISCC.exe YourScript.iss`. Expect lzma2/max compression of a
~640 MB game tree to produce roughly a 350 MB installer (actual ratio depends
heavily on how much of the content is already-compressed video/audio/texture
data).

**Verify before shipping:** run the compiled installer silently to a scratch
directory (`YourInstaller.exe /VERYSILENT /SUPPRESSMSGBOXES /DIR="<scratch
path>" /TASKS=""`), confirm the registry key landed correctly, confirm
`ce.exe` launches from that scratch install, then delete the scratch install
and re-point the registry `Path` back at your real install if you were testing
against a live setup.

---

## 5. Dead ends (what didn't work, and why)

| Attempt | Why it failed |
|---|---|
| Running the retail `SETUP.EXE` directly | SafeDisc-protected; `secdrv.sys` cannot load on 64-bit Windows (§3.1) |
| Running the 1.33 patch's `SETUP.EXE` directly | 16-bit NE executable; 64-bit Windows has no 16-bit subsystem at all (§3.2) |
| Setting only `HKCU` registry values (no admin) | The check reads `HKLM`, not `HKCU`; without elevation the key can't be written where the game looks |
| Setting `Status` as `REG_SZ "1"` | Wrong type (game reads it as a raw `DWORD`) and wrong value |
| Setting `Status` as `REG_DWORD` but with the value `966189` | Arithmetic error converting `0xEC62D` to decimal by hand (correct value: `968237`) — always convert hex constants in disassembly with a calculator/interpreter, not by hand |
| Trying `LOBBY.EXE` as an alternate "proper" launcher | It's a background multiplayer helper process spawned *by* `ce.exe`, not a front-end; run alone it exits silently with no window |
| Mounting the extracted ISO as a virtual CD-ROM drive and pointing `Drive` at it | Had no effect either way once the real `Status` fix was in place — the disassembled check never inspects `Drive` or any physical/virtual drive (§3.5) |
| Running as Administrator to fix the launch issue | Had no effect on `ce.exe` once the registry fix was in place; not required for this executable (§3.6) |
| Replicating official install behavior via the patch's real `SETUP.EXE` | Blocked entirely by the 16-bit-stub problem above; its install *script* (`setup.ins`) was still useful read as plain text/strings to confirm the exact registry key path the installer itself used |

---

## 6. Verification checklist

1. `<GAME_DIR>\ce.exe` exists and, when inspected (e.g. with any PE viewer),
   does **not** contain SafeDisc markers (`BoG_` string, `secdrv`/`drvmgt`
   references). If it does, you have the wrong/protected executable.
2. `HKLM\SOFTWARE\WOW6432Node\Take2\Codename Eagle\SrcDisk` exists with `Path`
   (string, your install dir), `Status` (**DWORD**, `968237`), and some value
   for `Drive`.
3. `<GAME_DIR>\DDraw.dll`, `D3DImm.dll`, `D3D8.dll`, `D3D9.dll`, and
   `dgVoodooCpl.exe` are present alongside `ce.exe`.
4. Launching `ce.exe` (directly, or via the launcher) should **not** show a
   "Codename: Eagle not installed" dialog. It should proceed into the game
   (expect an intro cinematic / main menu).

### Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "Codename: Eagle not installed" dialog, process exits | `Status` registry value missing, wrong type, or wrong value | Re-apply §4.7 exactly; confirm type is `REG_DWORD` and value is decimal `968237` |
| Windows says the file "can't run on this device" / mentions 16-bit | You're trying to run a 16-bit installer stub (`SETUP.EXE` from the original disc or the patch) | Don't run it; extract its `data1.cab` with `unshield` instead (§4.4/§4.6) |
| "Access Denied — Please login with administrator privileges" | You launched the SafeDisc-protected `Game.exe` instead of the unprotected `ce.exe` | Launch `ce.exe`, not `Game.exe` |
| 7-Zip reports "Cannot open the file as archive" on `data1.cab` | It's the legacy InstallShield "ISc(" cabinet format, not an MS-CAB | Use `unshield`, not 7-Zip, for any `data1.cab`/`.cab` from an InstallShield-based 1990s/2000s installer |
| 7-Zip reports "Cannot open the file as archive" on the raw `.img` | CloneCD images store raw 2352-byte sectors, not a plain ISO9660 image | De-raw the data track first (§4.2) before trying to browse it as a filesystem |
| No sound/music | Music is intentionally disabled by default in the launcher (per the owning user's preference in this session) | Uncomment the two VLC lines in the launcher `.bat`, or re-run the installer with the music task checked |
| Game launches but renders incorrectly / wrong resolution | dgVoodoo2 configuration | Run `dgVoodooCpl.exe` (GUI) or edit `dgVoodoo.conf` directly; see §4.8 for the settings used in this session |

---

## 7. For AI agents: key facts and pitfalls before you start

- **Identify the protection scheme before touching the installer.** Look for
  `secdrv.sys`, `clcd16/32.dll`, `dplayerx.dll`, `drvmgt.dll`,
  `00000001.TMP` → SafeDisc. These cannot be made to work on 64-bit Windows by
  any legitimate configuration change; the driver literally does not exist in a
  64-bit build. Do not attempt to install/crack this without the owner's
  explicit, informed decision — ask first.
- **`.img`/`.ccd`/`.cue`/`.sub` from CloneCD is not directly browsable.** The
  data track is raw 2352-byte sectors; strip to 2048-byte user data per sector
  (offset 16, length 2048) before any ISO9660 tool can read it. Compute the
  sector range from the `.cue` file's track timestamps, and cross-check against
  the `.ccd` file's TOC entries.
- **`data1.cab` from an InstallShield-based installer of this era is usually
  NOT a Microsoft CAB file.** Check the magic bytes: `ISc(` means you need
  `unshield`, not 7-Zip or any MS-CAB tool.
- **A `SETUP.EXE` from 1997–2000 is very often a 16-bit NE stub.** `file
  SETUP.EXE` will tell you. If so, it cannot run on 64-bit Windows under any
  circumstances (no NTVDM exists there) — don't waste time on compatibility
  mode flags for it. Go straight for its cabinet payload instead.
- **A game patch may silently ship a second, unprotected executable** alongside
  one that's still DRM-wrapped. Check PE section counts and search for DRM
  marker strings (`BoG_` for SafeDisc) in *every* top-level `.exe` a patch
  drops in before assuming you need to crack anything.
- **Black-box registry guessing has limits.** If an old game's own error
  message doesn't change despite registry changes that *should* affect it,
  that's a sign you're fixing the wrong check, or fixing it with the wrong
  type/value. Process Monitor (trace registry + file activity, filtered to the
  target process) is a legitimate, non-invasive way to see exactly what's being
  read and whether it succeeded — do this *before* reaching for a disassembler.
- **Disassembly is the last resort, and should require the user's explicit
  permission** (this session's environment flagged it as a sensitive action by
  default, and that's a reasonable default). When you do disassemble: resolve
  indirect `call dword ptr [addr]` targets against the PE's import table (don't
  guess), and **always convert hex constants with code, not mental arithmetic**
  — a single hand-conversion mistake (`0xEC62D` → wrong decimal) was the actual
  reason an otherwise-correct fix failed for several iterations in this
  session.
- **Not every registry value a community source tells you to set is actually
  checked.** This session set a `Drive` value based on an old forum post, then
  later proved (by removing a virtual CD-ROM drive entirely and re-testing)
  that the executable never reads it. Verify empirically where you can, and
  mark anything you can't verify as unverified rather than asserting it as
  fact.
- **When building a redistributable installer with Inno Setup for a 32-bit
  game**, do not set `ArchitecturesInstallIn64BitMode`. Leaving the installer
  as a 32-bit process makes its `HKLM\SOFTWARE` writes automatically redirect
  to `WOW6432Node`, which is where a 32-bit game process will actually look —
  matching registry views without any special-casing.
- **Test the installer itself**, not just the manually-assembled install — run
  it silently to a scratch directory and confirm the game launches from that
  output before calling the job done.
