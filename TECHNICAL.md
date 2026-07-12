# Technical write-up

How the Android Auto display glitch on the Bigme HiBreak / HiBreak Pro was traced and fixed.

## Symptom

Dark outlines / halos around every text and icon element on the Android Auto head-unit
display — like an over-sharpened JPEG. Worst immediately after opening a new app inside
Android Auto. Reproducible on real head units (Audi A3, Renault Koleos) **and** on the
Android Auto Desktop Head Unit (DHU) emulator with no car present, which ruled out the car,
cables, and head unit.

## Root cause

Bigme's `SurfaceFlinger` is patched with custom shaders, visible in logcat:

```
D surfaceflinger: createProcessShader:  colorMode=..,saturation=..,contrast=..,brightness=..
D surfaceflinger: createProcessShader2: textEnhance=..
```

`textEnhance` is an edge-darkening pass for the e-ink panel. It is applied to **all** display
layer stacks, including Android Auto's virtual display — so it gets composited into the video
stream sent to the car's LCD, where it appears as halos.

The value is chosen per foreground app by `xrz.framework.server.DisplayPolicyService`:

```
D DisplayPolicyService: Top app change from <a> to <b>
```

On each top-app change it applies that app's stored `app_text_enhance`. Observed values:

| App | text-enhance |
|-----|--------------|
| default (`ro.vendor.xrz.default_text_enhance`) | 70 |
| Settings | 50 |
| Android Auto (`com.google.android.projection.gearhead`) | 80 |
| Screensaver (`com.xrz.standby`) | 0 |

The screensaver is the only profile with 0, which is why **locking the phone** temporarily
cleared the glitch — it foregrounds the screensaver, whose value is 0.

There is **no display-scoping** in the code. `applyDisplayPolicy()` pushes one global value to
SurfaceFlinger, which applies it everywhere. The car display is an innocent bystander.

## Why the E-Ink Centre toggle does nothing

The per-app "Text Enhancement" toggle in E-Ink Centre does not write `app_text_enhance` on the
path SurfaceFlinger reads. Toggling it leaves the value unchanged (verified live: Spotify stays
70 on or off, Settings 50, Android Auto 80). The value that corrupts the projection is not
reachable from that UI.

## The fix

The value lives in a system-server database (`disp_policy.db`, table `policy_org`, column
`app_text_enhance`), which is root-only. But `DisplayPolicyService` exposes a binder method:

```java
public boolean setTextEnhanceForPackage(String packageName, int value)
```

with **no permission check** — callable from the `shell` UID. The interface
`IDisplayPolicyManager` (in `framework.jar`) numbers it transaction **34**. So:

```
adb shell service call xrz_display_policy_service 34 s16 <package> i32 0
```

writes the app's row to 0 and applies it immediately. `Result: Parcel(... 00000001 ...)` = true.
It persists across reboots (it's a real DB write, unlike the volatile
`vendor.xrz.global_text_enhance` property).

## Reading current values

`getPolicyByPackage` is transaction **1**; it returns the `DisplayPolicy` parcelable. The
effective text-enhance is the **second-to-last 32-bit integer** in the parcel dump (verified
against Settings=50, Chrome=70, Android Auto=0, Maps=0). The GUI uses this to display live
values.

## Re-deriving the transaction number (other firmware)

If transaction 34 stops working after a firmware update:

```
adb pull /system/framework/framework.jar
# decompile with jadx
grep -n "TRANSACTION_setTextEnhanceForPackage" .../xrz/framework/manager/IDisplayPolicyManager.java
```

The number printed there is the `service call` code.

## Residual issue

Because the glitch tracks the *whole* per-app profile leaking to the projected display, apps
that foreground during a drive (maps, dialer, in-call UI) can each re-trigger it until their
rows are also set to 0. There is also an intermittent, timing-dependent component that appears
related to the panel's transient refresh-mode state (`setLayerRefreshMode`) — this is not fully
fixable from outside the firmware.

## For Bigme

The correct fix is a small change in `DisplayPolicyService` / the SurfaceFlinger shader path:
apply the `textEnhance` and `colorMode` shaders **only to the primary physical display**
(display 0), never to virtual displays. EPD post-processing compensates for e-ink panel physics
and is meaningless — actively destructive — on a projected LCD, HDMI output, or screen
recording. This fixes the entire class of bug at once.

As a stopgap, Bigme could apply the same `textEnhance=0` treatment to
`com.google.android.projection.gearhead` that is already applied to `com.xrz.standby`.
