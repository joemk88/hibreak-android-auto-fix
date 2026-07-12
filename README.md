# HiBreak Android Auto Display Fix

Fixes the dark halo / edge-artifact glitch on the **Android Auto** display in your car when
using a **Bigme HiBreak** or **HiBreak Pro** e-ink phone.

![platform](https://img.shields.io/badge/platform-Windows%20%7C%20Linux%20%7C%20macOS-blue)
![root](https://img.shields.io/badge/root-not%20required-green)
![license](https://img.shields.io/badge/license-MIT-lightgrey)

---

## The problem

On the HiBreak / HiBreak Pro, Android Auto's projected display shows dark outlines / halos
around text and icons — it looks like an over-sharpened image. It's worst right after opening
a new app in the car (maps, dialer, media).

## The cause

Bigme's e-ink **text-enhancement** shader is applied to *all* displays, including the virtual
display that feeds the car head unit. It exists to make text pop on the e-ink panel, but on the
car's LCD it just corrupts the image. The display service applies whichever value the foreground
app carries, so every app that comes to the front while driving can re-trigger it.

Full technical write-up: see [`TECHNICAL.md`](TECHNICAL.md).

> **This is not the "Text Enhancement" toggle in E-Ink Centre.** That toggle does not change
> this value. This tool writes the underlying per-app display policy directly, through Bigme's
> own system service (`xrz_display_policy_service`).

## The fix

Set the text-enhance value to **0** for the apps you use while driving. It's **per app**,
**permanent**, and **survives reboots**. No root required.

---

## Download & use (Windows, easiest)

1. Grab **`HiBreak_AA_Fix.exe`** from the [Releases](../../releases) page.
2. On the phone: enable **Developer options** (tap Build number 7×) and turn on
   **USB debugging**. Plug in over USB and tap **Allow** on the prompt.
3. Double-click the exe. Android Auto is pre-ticked; tick any other apps you use while
   driving (phone/dialer, maps).
4. **Save backup**, then **Apply**. Unplug and drive.

Patched apps also look slightly lighter on the phone's own e-ink screen — only patch apps you
don't mind looking a little lighter. Use **Reset** on a row or **Restore from backup** to undo.

> Windows may show a SmartScreen warning for the unsigned exe ("More info → Run anyway"), and
> some antivirus tools flag PyInstaller exes. If you'd rather not trust the exe, run the Python
> source instead (below) — it's the same code.

---

## Run from source (any OS)

Requirements: **Python 3** (with tkinter) and **adb**.

- Linux: `sudo apt install python3-tk` if you get a tkinter error.
- adb must be on your PATH, or put `adb.exe` + `AdbWinApi.dll` + `AdbWinUsbApi.dll` next to
  the script.

```
python3 src/HiBreak_AA_Fix_GUI.py
```

---

## Manual command (no GUI)

For a single app:

```
adb shell service call xrz_display_policy_service 34 s16 <package.name> i32 0
```

`Result: Parcel(... 00000001 ...)` = success. To undo, run again with the original value
(Android Auto = 80, most apps = 70, Settings = 50):

```
adb shell service call xrz_display_policy_service 34 s16 <package.name> i32 70
```

---

## Build the exe yourself

```
pip install pyinstaller
```

Copy `adb.exe`, `AdbWinApi.dll`, `AdbWinUsbApi.dll` (from Google's SDK Platform Tools) next to
`src/HiBreak_AA_Fix_GUI.py`, then run `build_windows.bat`, or:

```
python -m PyInstaller --onefile --windowed --name "HiBreak_AA_Fix" ^
  --add-binary "adb.exe;." --add-binary "AdbWinApi.dll;." --add-binary "AdbWinUsbApi.dll;." ^
  src/HiBreak_AA_Fix_GUI.py
```

Result: `dist/HiBreak_AA_Fix.exe` (single standalone file, adb bundled inside).

---

## Firmware note

The tool uses transaction **34** (`setTextEnhanceForPackage`) on `xrz_display_policy_service`.
This is correct for the firmware it was built against; a future update could renumber it. If
Apply reports a failure (no `00000001`), see [`TECHNICAL.md`](TECHNICAL.md) for how to
re-derive the number from `framework.jar`.

---

## For Bigme

The correct fix is in firmware: in `DisplayPolicyService`, the text-enhance / colour shaders
should be applied only to the **primary physical display**, not to virtual displays. See
[`TECHNICAL.md`](TECHNICAL.md#for-bigme).

---

## License

MIT — see [`LICENSE`](LICENSE). Provided as-is; you run it at your own risk. Always save a
backup before applying.
