# Debloat an Android TV over Wi-Fi and replace its home screen

### A field-tested procedure for the Kodak PATH_7XPRO (Android 9 · Amlogic txlx · 1 GB RAM)

This is the exact procedure that was run on one Kodak/Thomson `PATH_7XPRO` TV and verified with
screenshots after every batch of changes. It removes the ad-heavy stock home screen, disables
telemetry and preloaded apps, and installs a clean launcher — **without root, without a USB cable,
and without uninstalling a single system package**.

You can follow it by hand, or hand this document to an AI coding agent and let it drive ADB. Both
routes are described below.

| | |
|---|---|
| Difficulty | Medium — you type commands, the TV stays usable the whole time |
| Time | 45–90 minutes, including verification |
| Risk | Low. Everything is a reversible `disable`/`enable` toggle |
| Root | Not required, not used |
| Data loss | None. No `pm uninstall` is ever run |

> **Read section 9 before you start.** Leaving ADB over the network enabled means anyone on your
> Wi-Fi can control the TV. The last step turns it off.

---

## 1. What you end up with

| Result | Detail |
|---|---|
| New home screen | **Plume Launcher 1.5** (`dev.plume`), survives cold boot |
| Ad-free YouTube | **SmartTube stable 32.68**, `armeabi-v7a` build (`org.smarttube.stable`) |
| 20 packages disabled (21 with Media Manager) | telemetry, recommendation rows, preloads, one-time setup helpers, OTA updater, stock launcher |
| Faster UI | no stock launcher, no recommendation service, no auto-install service |
| Everything reversible | one list of package names + `pm enable` |

**What actually gets copied onto the TV** (this is the complete list — nothing else is installed):

| File | Package | Size |
|---|---|---|
| `Plume-1.5.apk` | `dev.plume` | ~0.4 MB |
| `SmartTube_stable_32.68_armeabi-v7a.apk` | `org.smarttube.stable` | ~25 MB |

Everything else in this guide is a settings change made through `adb shell`.

---

## 2. What is kept (do not disable these)

These were explicitly preserved and tested after the cleanup:

- **Live TV / Dish set-top box** — `com.android.tv` (note: this is *not* `com.google.android.tv`)
- **HDMI inputs and tuner** — DroidLogic TV input services
- **Remote, keyboard, Bluetooth** — input and Bluetooth stack
- **Google Cast / Chromecast** — `com.google.android.apps.mediashell`
- **Miracast / screen mirroring**
- **Voice assistant** — `com.google.android.katniss`, `com.google.android.tts` / `speech.pumpkin`
- **Play Store + Google Play services**
- **YouTube** (stock app left installed), **JioHotstar**, **VLC**
- **File Manager +**, **Send Files to TV**
- **Amlogic Photo / Music / Video players**
- **MediaProvider / ExternalStorageProvider** — needed by galleries, VLC and file managers

---

## 3. Prerequisites

**Hardware**

- The TV, connected to the same network as your computer (2.4 GHz works, 5 GHz is better)
- The TV remote (you will have to approve one on-screen dialog)
- A computer with a terminal

**Software**

- `adb` (Android platform-tools). On Windows:
  ```powershell
  winget install --id Google.PlatformTools -e
  ```
  This does **not** add `adb` to `PATH`. Either add
  `%LOCALAPPDATA%\Microsoft\WinGet\Packages\Google.PlatformTools_*\platform-tools` to `PATH`,
  or call `adb.exe` by full path. All commands below assume `adb` is callable.
- Optional: Python 3 (only for the port-closed check in section 9)
- Optional: Windows 10/11 for the OCR helper described in section 7

**Knowledge**

- How to find the TV's IP address: *Settings → Network & Internet → your network → IP address*
- How to type `adb` commands

**Set a variable for the rest of the session**

PowerShell:
```powershell
$TV = "<TV_IP>:5555"
```
Bash / macOS / Linux:
```bash
export TV="<TV_IP>:5555"
```

---

## 4. The target device

Recorded here because the quirks below are specific to this firmware generation.

| Item | Value |
|---|---|
| Model | `PATH_7XPRO` |
| Brand string | `THOMSON_KODAK` |
| Build / device | `LEDTV32HD` (Android 9, SDK 28) |
| SoC | Amlogic `txlx` |
| CPU ABI | **`armeabi-v7a` only** (32-bit — arm64 APKs will not install) |
| Display | 1280×720, 213 dpi |
| RAM | ~1 GB (`MemTotal ≈ 1014772 kB`) |
| Storage | 4.2 GB internal |

Two consequences that matter:

1. **Always pick the `armeabi-v7a` APK.** Check with:
   ```powershell
   adb -s $TV shell getprop ro.product.cpu.abilist
   ```
2. **1 GB of RAM is the real performance limit.** A single video app left playing in the
   background is enough to make the whole TV feel slow. See section 10.

---

## 5. Manual procedure

### 5.1 Enable debugging on the TV

1. *Settings → Device Preferences → About*
2. Tap **Build** seven times → "You are now a developer"
3. *Settings → Device Preferences → Developer options*
4. Turn **USB debugging ON**. On this firmware that also opens ADB over TCP port 5555.
   (Android 9 has no "Wireless debugging" toggle — that is Android 11+.)

### 5.2 Connect

```powershell
adb connect $TV
adb devices
```

The first time, the TV shows **"Allow USB debugging?"**. Tick *Always allow from this computer* →
**Allow**. `adb devices` must now show `device` (not `unauthorized`).

Sanity check:

```powershell
adb -s $TV shell getprop ro.product.model
adb -s $TV shell getprop ro.build.version.release
```

### 5.3 Back up before changing anything

Non-negotiable. These are all read-only.

```powershell
mkdir tv-backup
cd tv-backup

adb -s $TV shell getprop > device-info.txt
adb -s $TV shell pm list packages > packages-all.txt
adb -s $TV shell pm list packages -u > packages-including-uninstalled.txt
adb -s $TV shell pm list packages -d > packages-disabled-before.txt
adb -s $TV shell settings list global  > settings-global.txt
adb -s $TV shell settings list secure  > settings-secure.txt
adb -s $TV shell settings list system  > settings-system.txt
adb -s $TV shell wm size     > screen-size.txt
adb -s $TV shell wm density  > screen-density.txt

adb -s $TV shell "cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME" > home-before.txt
adb -s $TV shell settings get global private_dns_mode      > dns-mode-before.txt
adb -s $TV shell settings get global private_dns_specifier > dns-specifier-before.txt

adb -s $TV shell screencap -p /sdcard/before.png
adb -s $TV pull /sdcard/before.png .
```

If `adb exec-out screencap -p > before.png` produces a 0-byte file (it does on this firmware right
after boot), use the `screencap` + `pull` pair above instead.

Create your own change log now — you will thank yourself in section 8:

```powershell
"disabled-packages:" | Out-File -Append change-log.txt
```

### 5.4 Disable packages — one at a time, verified

The rule: **`pm disable-user`, never `pm uninstall`.**

```powershell
adb -s $TV shell pm disable-user --user 0 <package.name>
adb -s $TV shell pm list packages -d | findstr <package.name>   # must print the package
```

After each *batch* (not each package), take a screenshot and confirm the home screen is still
alive:

```powershell
adb -s $TV shell "dumpsys window | grep mCurrentFocus"
```

The 20 packages (21 with the optional Media Manager), grouped in the order that was actually used.
Start with the safe groups; keep group 5 for last.

**Batch 1 — telemetry**

| Package | Why |
|---|---|
| `com.google.android.feedback` | Google Feedback telemetry |
| `com.google.android.tv.bugreportsender` | bug-report sender |

**Batch 2 — recommendations and store**

| Package | Why |
|---|---|
| `com.google.android.tvrecommendations` | sponsored recommendation rows |
| `com.google.android.leanbacklauncher.recommendations` | stock-launcher recommendation service |
| `com.google.android.tv` | Google TV store/recommendations app |

**Batch 3 — preloads**

| Package | Why |
|---|---|
| `com.amazon.amazonvideo.livingroom` | Prime Video preload |
| `com.sonyliv` | SonyLIV preload |
| `com.tcl.browser` | TCL "BrowseHere" ad-portal browser |
| `com.sppl.emanual` | e-manual |

**Batch 4 — unused background services**

| Package | Why |
|---|---|
| `com.dsi.ant.usbservice` | ANT+ USB service (no ANT accessories) |
| `com.android.printspooler` | print spooler (no printing on a TV) |
| `com.android.smspush` | WAP-push / carrier SMS service |
| `com.cultraview.agingmodel` | factory aging / burn-in test app |

**Batch 5 — one-time setup helpers + OTA**

| Package | Why |
|---|---|
| `com.google.android.partnersetup` | one-time Google partner setup |
| `com.google.android.onetimeinitializer` | one-time initializer |
| `com.android.onetimeinitializer` | one-time initializer |
| `com.google.android.tvsetup.partnercustomizer` | one-time TV setup customizer |
| `android.autoinstalls.config.amlogic.device` | silent app auto-install config |
| `com.droidlogic.otaupgrade` | OTA firmware updater |

> Disabling the OTA updater is a deliberate trade-off: no more firmware or security updates until
> you re-enable `com.droidlogic.otaupgrade`. Decide for yourself.

**Batch 6 — stock launcher (do this in 5.5)**

| Package | Why |
|---|---|
| `com.google.android.tvlauncher` | stock Google TV launcher, replaced by Plume |

Optionally also: `com.yunfang.files` ("Media Manager" preload) if you do not use it.

### 5.5 Install Plume and hand over Home

```powershell
adb -s $TV install -r Plume-1.5.apk
adb -s $TV shell pm grant dev.plume android.permission.WRITE_SECURE_SETTINGS
adb -s $TV shell pm grant dev.plume android.permission.READ_EXTERNAL_STORAGE
adb -s $TV shell am start -n dev.plume/.MainActivity
```

**Look at the TV.** Plume asks *"Allow Plume to read all TV listings?"* → **Allow**.
This dialog cannot be accepted over ADB: `pm grant` does not dismiss it, and `KEYCODE_BACK` does
not close it. It has to be the remote.

Then:

```powershell
adb -s $TV shell cmd package set-home-activity --user 0 dev.plume/.MainActivity
adb -s $TV shell am start -W -a android.intent.action.MAIN -c android.intent.category.HOME -n dev.plume/.MainActivity
adb -s $TV shell pm disable-user --user 0 com.google.android.tvlauncher
adb -s $TV shell input keyevent 3
```

> **Firmware quirk — read this.** `set-home-activity` prints `Success` and PackageManager even logs
> `Replacing preferred activity dev.plume/.MainActivity`, but pressing Home still launches
> `com.google.android.tvlauncher`. The handover only completes when the stock launcher is
> **disabled**. That is why the disable step above is part of the procedure and not optional.
>
> After a reboot, `resolve-activity` can still name the stock launcher for a few seconds while
> Plume is already on screen. Trust `mCurrentFocus` **plus** a screenshot, not `resolve-activity`.

### 5.6 SmartTube (ad-free YouTube)

```powershell
adb -s $TV install -r SmartTube_stable_32.68_armeabi-v7a.apk
adb -s $TV shell monkey -p org.smarttube.stable -c android.intent.category.LAUNCHER 1
```

Notes:

- The package id is **`org.smarttube.stable`** — not `com.liskovsoft.*`. Looking up the wrong id
  returns "No activities found to run" and looks like a broken install.
- The stock YouTube app stays installed. If you want it gone from the app list, disable it
  (reversible) — do not uninstall it.

### 5.7 Optional — Private DNS

If you want encrypted DNS (and, with a filtering resolver, some ad/tracker blocking):

```powershell
adb -s $TV shell settings put global private_dns_specifier dns.google
adb -s $TV shell settings put global private_dns_mode hostname
```

Verify:

```powershell
adb -s $TV shell dumpsys connectivity | findstr /i "PrivateDns"
# UsePrivateDns: true   PrivateDnsServerName: dns.google
```

Notes:

- DoT resolvers add latency per uncached lookup. Measure before committing to a filtering
  resolver: on this TV, `dns.google` answered in ~13 ms while a filtering resolver took ~85 ms.
- DNS filtering blocks trackers and banner ads, **not** video ads.
- The original values on this TV were unset. If yours are set, record them (5.3) before changing.

### 5.8 Prove it survives a reboot

```powershell
adb -s $TV reboot
# wait ~60 s, then:
adb connect $TV
adb -s $TV shell getprop sys.boot_completed     # 1
adb -s $TV shell input keyevent 3
adb -s $TV shell "dumpsys window | grep mCurrentFocus"
adb -s $TV shell "cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME"
```

Both the focused window and the resolved HOME activity should be `dev.plume/...`. ADB normally
comes back on its own after the reboot; if not, `adb connect` again.

---

## 6. Verification checklist

| Check | Command | Expected |
|---|---|---|
| ADB online | `adb devices` | `device` |
| Plume is running | `adb -s $TV shell "dumpsys window \| grep mCurrentFocus"` | `dev.plume/...MainActivity` |
| Plume is the resolved Home | `adb -s $TV shell "cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME"` | `dev.plume/...` |
| App installed | `adb -s $TV shell pm list packages \| findstr dev.plume` | `package:dev.plume` |
| SmartTube installed | `adb -s $TV shell pm list packages \| findstr smarttube` | `package:org.smarttube.stable` |
| Disabled list | `adb -s $TV shell pm list packages -d` | your packages + factory-disabled ones |
| Live TV still works | open Live TV on the TV | Dish/HDMI picture |
| Cast still works | cast from a phone | TV appears in the device list |
| Free space | `adb -s $TV shell df /data` | enough headroom for future installs |
| RAM headroom | `adb -s $TV shell "cat /proc/meminfo \| head -4"` | `MemAvailable` in the hundreds of MB |

Screenshot proof (keep these, they are the only reliable evidence):

```powershell
adb -s $TV shell screencap -p /sdcard/after.png
adb -s $TV pull /sdcard/after.png .
```

---

## 7. Doing it with an AI agent

This is a good task for an agent because it is long, repetitive and needs constant verification.
Give the agent this document and a prompt like the one below. Replace the placeholders.

```text
You are cleaning up an Android TV over ADB. The TV is at <TV_IP>:5555 and the
procedure to follow is in ANDROID-TV-DEBLOAT-KODAK-PATH7XPRO.md in this repo.

Rules you must follow for the whole session:
1. NEVER run `pm uninstall`, `pm uninstall --user 0`, or anything requiring root.
   Use only `pm disable-user --user 0` and `pm enable`.
2. Before changing anything, create a `tv-backup/` folder and save: getprop,
   pm list packages (normal and -u), pm list packages -d, settings list
   global/secure/system, wm size, wm density, current HOME activity, and a
   screenshot.
3. Work in the batches in section 5.4. After each batch, verify the TV still has
   a working home screen: check `dumpsys window | grep mCurrentFocus` AND take a
   screenshot and look at it. Do not continue if the screen is broken.
4. Maintain a file `disabled-packages.txt` with one package per line. Every
   `disable-user` you run must be appended there in the same turn.
5. Ask me before: disabling the OTA updater, disabling the stock YouTube app,
   changing DNS, or doing anything not listed in the document.
6. The on-screen "Allow TV listings" dialog cannot be accepted over ADB. Stop and
   ask me to press Allow with the remote when you reach it.
7. Never guess UI coordinates. Take a screenshot, read it, then tap.
8. Report after every batch: packages disabled, mCurrentFocus, free RAM, and
   whether the home screen looks correct.
9. Finish by running the verification checklist in section 6 and writing a short
   summary of what changed and how to undo it.
10. Do not disable anything in the "keep" list in section 2.
```

**Why the verification loop matters.** On this firmware:

- `uiautomator dump` fails on system permission dialogs (`null root node returned by
  UiTestAutomationBridge`), so UI automation alone is not enough — screenshot + OCR works.
- `set-home-activity` lies (prints `Success`, changes nothing), so an agent that only checks
  command output will report a false success.
- `KEYCODE_MEDIA_PAUSE` pauses background media *and* brings that app to the front; always follow
  it with `KEYCODE_HOME` and re-check the focused window.
- `adb exec-out screencap` can return 0 bytes right after boot; use the file + `pull` method.
- PowerShell expands `$var` inside double-quoted `adb shell "..."` strings. Use single quotes for
  remote loops: `adb shell 'for h in a b; do ping $h; done'`.

A small OCR helper makes screenshot-reading reliable on Windows (Windows.Media.Ocr, no extra
install):

```powershell
# ocr.ps1 -Text C:\path\to\screen.png        -> plain text
# ocr.ps1 -Text -Boxes C:\path\to\screen.png -> text + tap coordinates
```

---

## 8. Undo everything

Re-enable package by package:

```powershell
adb -s $TV shell pm enable com.google.android.tv
```

Batch restore from your list (PowerShell):

```powershell
Get-Content disabled-packages.txt | ForEach-Object {
  if ($_ -and $_ -notmatch '^#') { adb -s $TV shell pm enable $_ }
}
```

Order matters for the launcher:

1. `pm enable com.google.android.tvlauncher`
2. Set Home back to the stock launcher:
   `adb -s $TV shell cmd package set-home-activity --user 0 com.google.android.tvlauncher/.MainActivity`
3. Optionally disable Plume: `adb -s $TV shell pm disable-user --user 0 dev.plume`
4. Optionally restore the original DNS values you saved in 5.3:
   `adb -s $TV shell settings delete global private_dns_specifier` (if the original was unset)
5. Reboot.

Also on the TV: *Settings → Apps → See all apps → [app] → Enable*. Disabled apps always remain
listed there; they just lose their launcher tile.

---

## 9. When you are done — close network ADB

While USB debugging is on, port 5555 is open on your network. Close it:

```powershell
adb -s $TV shell settings put global adb_enabled 0
adb -s $TV disconnect $TV
adb kill-server
```

Verify from the computer (should fail / refuse):

```powershell
Test-NetConnection <TV_IP> -Port 5555
```

Or with Python:

```python
python -c "import socket,sys; s=socket.socket(); s.settimeout(4); sys.exit(0 if s.connect_ex(('<TV_IP>',5555))!=0 else 1)"
# exit 0 = port closed, exit 1 = still open
```

Manual fallback if the commands do not stick (disabling debugging can take effect after a reboot):

1. *Settings → Device Preferences → About* → tap **Build** 7×
2. *Developer options* → **USB debugging OFF**, and any **Network debugging / ADB over network** or
   **Wireless debugging** toggle OFF
3. Reboot, then check the port again

To debug again later, turn USB debugging back on and `adb connect <TV_IP>:5555`.

---

## 10. Troubleshooting (symptoms seen on real hardware)

| Symptom | Cause | Fix |
|---|---|---|
| `set-home-activity` says `Success`, Home key still opens the stock launcher | firmware ignores the preferred-activity change | disable `com.google.android.tvlauncher` |
| `uiautomator dump` → `null root node` | system dialogs are not exposed to UiAutomator | screenshot + OCR, then tap |
| `exec-out screencap` writes a 0-byte PNG | capture raced the boot transition | `screencap -p /sdcard/x.png` then `adb pull` |
| Music/media pull fails with `Not a directory` | `adb pull` on Windows cannot create folders with emoji in the name | rename folders to ASCII **on the TV**, pull, rename back |
| Home key "sticks" in an app | a modal/sign-in screen still owns focus | `am force-stop <pkg>`, then `KEYCODE_HOME` |
| `monkey -p com.liskovsoft...` → no activities | wrong package id | use `org.smarttube.stable` |
| `INSTALL_FAILED_INSUFFICIENT_STORAGE` | ~4 GB internal, very little free | remove large files, use the smallest APK variant |
| ADB unreachable after a router reboot | TV IP is DHCP | check the TV's IP, or add a DHCP reservation |
| Everything feels slow while idle | a video app is playing in the background | `dumpsys media_session`, pause, `am force-stop` it |
| `adb shell "for x in a b; do ping $x; done"` pings nothing | PowerShell expanded `$x` locally | use single quotes around the remote command |
| TV shows `unauthorized` | debugging prompt not accepted | accept "Always allow" on the TV, `adb kill-server`, reconnect |
| `where adb` finds nothing after winget | platform-tools is not added to `PATH` | call the full path or add the folder to `PATH` |

**The 1 GB RAM problem.** On this TV `MemTotal ≈ 1 GB`. During one "the internet is slow" report,
only ~54 MB was free with ~164 MB of swap in use — because the stock YouTube app was still
*playing* in the background behind the launcher, holding ~125 MB and ~29% CPU. Pausing and
force-stopping it took `MemAvailable` from ~188 MB to ~367 MB. Before blaming Wi-Fi, check:

```powershell
adb -s $TV shell "dumpsys media_session | grep -E 'package=|state=PlaybackState'"
adb -s $TV shell "cat /proc/meminfo | head -4"
```

Also close file managers you are not using: they are heavy for what they do.

---

## 11. Do not do this

| Don't | Why |
|---|---|
| `pm uninstall` / `pm uninstall --user 0` on a preload | irreversible without a factory reset; `disable-user` achieves the same result |
| Disable `com.android.tv` | that is **Live TV**; `com.google.android.tv` is the store app |
| Disable `MediaProvider` / `ExternalStorageProvider` | breaks gallery, VLC, file managers and casting |
| Disable `com.google.android.katniss` or the TTS engine | that is the voice assistant |
| Disable Google Play services / Play Store | breaks casting, sign-in and most apps |
| Run factory-test apps (`com.tvfactory`, `com.ctv.factorytest`) | no benefit, real risk |
| Use modded streaming APKs to remove video ads | malware / account-ban risk; no safe method exists |
| Root the TV | nothing in this procedure needs it |
| Leave ADB on port 5555 enabled | anyone on the network can control the TV |

---

## 12. Notes on scope

- Verified on **one** device: Kodak `PATH_7XPRO`, `THOMSON_KODAK`, build `LEDTV32HD`, Android 9.
  Other Amlogic/Android 9 TVs will be close but not identical — always take the backup first and
  test after each batch.
- Package names differ between brands. If a package in section 5.4 does not exist on your TV,
  `pm disable-user` will just say "Unknown package" — skip it.
- Numbers like free RAM, free storage and DNS latency are measurements from this unit, not
  guarantees.

Share, fork and improve freely.
