# Vwar-S13-Core5-Root-Guide
Root Guide for a legit S13/Core 5 watch running Android 13
# Vwar S13 / Core 5 Root Guide

A working root method for the **Vwar S13 / Core 5 full-Android smartwatch** running Android 13 on the **ASR8601** platform.

> [!IMPORTANT]
> This procedure was developed and tested on one personally owned S13/Core 5. These watches may ship with different firmware even when sold under the same model name. **Dump your own partitions before flashing anything.**
>
> Do not blindly flash images from this guide or from another watch.

## Tested Device

| Property | Tested value |
|---|---|
| Model | S13 |
| Platform / SoC | ASR8601 |
| Android device | `dove` |
| Product | `YBD_C29_OLED` |
| Android | 13 |
| SDK | 33 |
| Security patch | 2024-04-05 |
| Build | `S13_C29_EN_V1.6_20251121` |
| FOTA version | `S13_C29_EN_V1.6_20251121_20251121-1207` |
| Update system | A/B |
| Tested active slot | `_a` |
| Bootloader | Locked |
| AVB | 1.2 |
| Root | Magisk |

## AI Disclosure

This project was developed with extensive assistance from **OpenAI's ChatGPT**.

I performed the actual testing on the hardware, executed the commands, dumped and flashed the device, verified the results, and accepted the risk of modifying my own device.

ChatGPT was used extensively throughout the process to help analyze device and firmware information, inspect the Android/AVB boot chain, interpret command output, troubleshoot ASRClientCore, generate commands and scripts, reason through the rooting method, and help write this documentation.

In other words: **the hardware testing and verification were real, but AI was a major part of the research and reverse-engineering workflow.**

---

# Overview

The tested S13 has a **locked bootloader**, and normal Android/Fastboot methods do not provide access to `init_boot`.

The successful method was:

1. Access the ASR8601 low-level USB download interface.
2. Dump the stock `init_boot_a` and other critical partitions using ASRClientCore.
3. Patch the stock `init_boot_a` with Magisk.
4. Inspect the device's AVB trust chain.
5. Rebuild/sign the patched `init_boot` so it remains valid under the existing AVB chain.
6. Write the signed image back through the ASR8601 download interface.
7. Read the partition back and verify it byte-for-byte.
8. Boot Android and confirm Magisk root.

**The bootloader remained locked.**

Only `init_boot_a` was modified on the tested device.

---

# Why Normal Rooting Methods Didn't Work

The tested S13 reported:

```text
sys.oem_unlock_allowed=0
```

Attempting to dump `init_boot` from normal Android:

```bash
dd if=/dev/block/by-name/init_boot_a of=/sdcard/Download/init_boot_a.img
```

returned:

```text
Permission denied
```

Fastboot could see the partition size, but attempting:

```text
fastboot fetch init_boot_a
```

failed with:

```text
FAILED (remote: 'fetch init_boot_a partition is not allowed')
```

The important discovery was that the **ASR8601 exposes a lower-level USB download interface while the watch is powered off**.

That interface operates below normal Android and therefore bypasses the Android userspace/SELinux restrictions that prevented us from reading the partition.

---

# Requirements

You will need:

- Windows PC
- USB data cable compatible with the watch
- ADB/Fastboot Platform Tools
- .NET 8
- ASR USB drivers
- ASRClientCore
- Magisk
- `avbtool`
- Python if required by your AVB tooling

ASRClientCore:

https://github.com/YC-nw/ASRClientCore

Magisk:

https://github.com/topjohnwu/Magisk

Android AVB documentation:

https://android.googlesource.com/platform/external/avb/

---

# Pre-requisite:

Follow the 4 prong data cable hardware mod to allow the S13 to connect via USB (PDF guide attached to repo). 

# Step 1 — Identify Your Device

Enable USB debugging and connect the running watch.

Open PowerShell in your Platform Tools directory:

```powershell
.\adb devices

.\adb shell "getprop ro.product.model"
.\adb shell "getprop ro.product.device"
.\adb shell "getprop ro.product.name"
.\adb shell "getprop ro.product.board"
.\adb shell "getprop ro.hardware"

.\adb shell "getprop ro.build.display.id"
.\adb shell "getprop ro.build.version.release"
.\adb shell "getprop ro.build.version.sdk"
.\adb shell "getprop ro.build.version.security_patch"

.\adb shell "getprop ro.boot.slot_suffix"
.\adb shell "getprop ro.build.ab_update"
```

My tested device returned values corresponding to:

```text
Model: S13
Device: dove
Product: YBD_C29_OLED
Platform: ASR8601
Android: 13
SDK: 33
Security patch: 2024-04-05
Build: S13_C29_EN_V1.6_20251121
Active slot: _a
A/B updates: true
```

If your build differs, **do not assume my patched image or AVB parameters are valid for your device.**

---

# Step 2 — Inspect the Partition Layout

Run:

```powershell
.\adb shell "ls -l /dev/block/by-name"
```

The tested watch contains:

```text
boot_a
boot_b
init_boot_a
init_boot_b
vendor_boot_a
vendor_boot_b
vbmeta_a
vbmeta_b
vbmeta_system_a
vbmeta_system_b
```

On my device:

```text
init_boot_a -> /dev/block/mmcblk0p20
init_boot_b -> /dev/block/mmcblk0p21
boot_a      -> /dev/block/mmcblk0p22
```

The `init_boot` partitions were:

```text
0x800000 bytes
```

which is:

```text
8 MiB
```

---

# Step 3 — Prepare ASRClientCore

Clone ASRClientCore:

```powershell
git clone https://github.com/YC-nw/ASRClientCore.git
cd ASRClientCore
```

The version used during testing was built as **x86 Release** because of its wrapper/dependency configuration.

The resulting executable was:

```text
ASRClientCore\bin\x86\Release\net8.0-windows7.0\ASRClientCore.exe
```

## Important ASRClientCore Note

During testing, the upstream test program contained a bogus test erase operation referencing:

```text
fuck_you_asr
```

I disabled that test operation before using the program against the watch.

I **did not** disable the normal ASR handshake or keepalive behavior.

In fact, during troubleshooting I temporarily modified the handshake/keepalive behavior and partition reads stopped working. Restoring the upstream `FlashManager.cs` and `RequestManager.cs` behavior fixed communication.

The configuration that ultimately worked was therefore:

```text
Normal upstream ASR handshake
+
Normal upstream keepalive
+
Bogus test erase disabled
```

---

# Step 4 — Understand ASR Download Mode

The running Android device normally appears as approximately:

```text
VID_2ECC&PID_2004
```

The powered-off low-level ASR download interface appears as:

```text
VID_2ECC&PID_0707
```

with:

```text
ASR001
```

The important part is that PID `0707` can appear only briefly if a client does not grab it.

## Reliable Connection Procedure

1. Power the watch completely off.
2. Start ASRClientCore **before reconnecting the watch**.
3. Tell ASRClientCore to wait for the device.
4. Connect the powered-off watch.
5. ASRClientCore should catch the PID `0707` interface.

A successful connection looks similar to:

```text
Waiting for device connecting (120s). Connect your device directly just after powering down

device path:
\\?\usb#vid_2ecc&pid_0707#asr001#
```

---

# Step 5 — Dump Your Stock Firmware

**Do this before writing anything.**

Create a dump directory and read the important partitions:

```powershell
$exe = "C:\Users\YOURNAME\ASRClientCore\bin\x86\Release\net8.0-windows7.0\ASRClientCore.exe"
$out = "$env:USERPROFILE\Downloads\S13_STOCK_DUMP"

New-Item -ItemType Directory -Force $out | Out-Null

& $exe --wait 120 --timeout 30000 `
    r init_boot_a "$out\init_boot_a.img" `
    r boot_a "$out\boot_a.img" `
    r vbmeta_a "$out\vbmeta_a.img" `
    r vbmeta_system_a "$out\vbmeta_system_a.img" `
    r vendor_boot_a "$out\vendor_boot_a.img"
```

Once the command is waiting:

1. Make sure the watch is completely powered off.
2. Connect it to the PC.
3. Let ASRClientCore catch the ASR download interface.

A successful dump should show each partition reaching:

```text
100%
```

## Hash Your Backups

Immediately calculate SHA-256 hashes:

```powershell
Get-ChildItem "$out\*.img" |
    Get-FileHash -Algorithm SHA256
```

Keep these stock images somewhere safe.

**Never modify your only stock copy.**

---

# Step 6 — Patch `init_boot_a` With Magisk

Android 13 devices using `init_boot` should have the appropriate `init_boot` image patched rather than blindly patching `boot.img`.

Push your stock image to the watch:

```powershell
.\adb push "C:\path\to\init_boot_a.img" /sdcard/Download/init_boot_a.img
```

Open Magisk on the watch.

Select:

```text
Install
→ Select and Patch a File
→ init_boot_a.img
```

Wait for Magisk to finish.

Then pull the generated image back to the PC:

```powershell
.\adb pull /sdcard/Download/magisk_patched-XXXXX.img "$env:USERPROFILE\Downloads\S13_magisk_patched.img"
```

The exact Magisk-generated filename will vary.

---

# Step 7 — DO NOT Flash the Raw Magisk Image Yet

This is important.

The stock `init_boot_a` on my S13 was AVB protected.

Magisk modifies the ramdisk, so the original signed hash descriptor no longer describes the modified contents.

Therefore:

```text
Stock init_boot
        ↓
Magisk patch
        ↓
Old AVB metadata no longer matches payload
```

The raw Magisk output was **not** what I flashed.

It first had to be rebuilt/signed correctly for the AVB chain used by this firmware.

---

# Step 8 — Inspect the AVB Trust Chain

Use `avbtool` to inspect your own images.

For example:

```bash
avbtool info_image --image init_boot_a.img
```

Also inspect:

```text
vbmeta_a.img
vbmeta_system_a.img
```

On my tested firmware, `vbmeta_a` explicitly chained:

```text
boot
init_boot
vbmeta_system
```

The `init_boot` chain trusted an RSA-2048 key.

The public-key SHA-256 I observed for the stock `init_boot` was:

```text
22de3994532196f61c039e90260d78a93a4c57362c7e789be928036e80b77c8c
```

That matched the standard AOSP RSA-2048 AVB test key.

The top-level `vbmeta_a` also used an AOSP test key, in that case RSA-4096.

This was the critical discovery.

The bootloader did not need to be unlocked because the existing AVB chain already trusted the key that could be used to sign the modified `init_boot`.

## This is firmware-specific.

Verify the key on your own firmware.

Do not assume another S13/Core 5 uses the same AVB configuration.

---

# Step 9 — Rebuild and Sign the Magisk-Patched `init_boot`

Before modifying anything, record the AVB information from your stock image:

```bash
avbtool info_image --image init_boot_a.img
```

On my tested firmware, important parameters included:

```text
Algorithm: SHA256_RSA2048
Partition name: init_boot
Partition size: 8388608
Rollback index: 1712275200
```

The final image still needed to be exactly:

```text
8388608 bytes
```

or:

```text
8 MiB
```

Preserve the relevant stock AVB configuration and descriptors while generating metadata describing the **new Magisk-patched payload**.

Sign it using the key trusted by your firmware's parent `vbmeta` chain.

## Before Flashing, Verify:

- The image is exactly the correct partition size.
- The AVB footer is valid.
- The hash descriptor matches the modified payload.
- The RSA signature verifies.
- The public key matches the key trusted by `vbmeta_a`.
- The partition name remains `init_boot`.
- Appropriate rollback metadata is preserved.
- Any relevant stock AVB properties are preserved.

**Do not flash the image if these checks fail.**

---

# Step 10 — Flash Using READ → WRITE → READ

I did **not** erase `init_boot_a` first.

Instead, I used:

```text
READ CURRENT PARTITION
        ↓
WRITE PATCHED PARTITION
        ↓
READ PARTITION BACK
        ↓
VERIFY HASH
        ↓
REBOOT
```

Power the watch completely off.

Then run:

```powershell
$exe = "C:\Users\YOURNAME\ASRClientCore\bin\x86\Release\net8.0-windows7.0\ASRClientCore.exe"

$patched = "$env:USERPROFILE\Downloads\S13_magisk_AVB_signed.img"
$before  = "$env:USERPROFILE\Downloads\S13_INITBOOT_BEFORE_FLASH.img"
$after   = "$env:USERPROFILE\Downloads\S13_INITBOOT_AFTER_FLASH.img"

& $exe --wait 120 --timeout 30000 `
    r init_boot_a "$before" `
    w init_boot_a "$patched" `
    r init_boot_a "$after" `
    rst 0x15
```

Once ASRClientCore is waiting, connect the powered-off watch.

The successful process looked like:

```text
READ init_boot_a ... 100%
WRITE init_boot_a ... 100%
READ init_boot_a ... 100%
rebooting device to ColdRebootToNormal mode
```

---

# Step 11 — Verify the Flash

Compare the image you intended to flash with the image read back from the device:

```powershell
Get-FileHash `
    "$env:USERPROFILE\Downloads\S13_magisk_AVB_signed.img", `
    "$env:USERPROFILE\Downloads\S13_INITBOOT_AFTER_FLASH.img" `
    -Algorithm SHA256
```

The two SHA-256 hashes should be identical.

On my watch, the readback matched the image I wrote **byte-for-byte**.

That confirmed the flash completed correctly.

---

# Step 12 — Boot Android

After the cold reboot:

```powershell
.\adb wait-for-device

.\adb shell "getprop sys.boot_completed"
.\adb shell "getprop ro.boot.slot_suffix"
.\adb shell "getprop ro.build.display.id"
```

My watch returned:

```text
sys.boot_completed = 1
slot = _a
build = S13_C29_EN_V1.6_20251121
```

Android booted normally.

---

# Step 13 — Verify Root

Open Magisk on the watch.

Then run:

```powershell
.\adb shell "su -c 'echo ROOT_OK; id; whoami; magisk -v; getprop ro.boot.slot_suffix' 2>&1"
```

Magisk should display a Superuser request on the watch.

Approve it.

Successful root should produce output containing something similar to:

```text
ROOT_OK
uid=0(root)
root
...
_a
```

I additionally verified the device using Root Checker, which reported:

```text
Congratulations!
Root access is properly installed on this device!
```

At this point the S13 was successfully rooted.

---

# Restoring Stock

This is why the stock dump is important.

If you need to restore the original `init_boot_a`, use ASRClientCore in the same PID `0707` download mode.

Use your own original:

```text
init_boot_a.img
```

and write it back to:

```text
init_boot_a
```

Use the same procedure:

```text
READ
 ↓
WRITE STOCK IMAGE
 ↓
READ BACK
 ↓
VERIFY SHA-256
 ↓
REBOOT
```

Never restore an `init_boot` taken from an unknown firmware revision.

---

# A/B OTA Warning

The S13 tested here uses A/B updates.

I modified:

```text
init_boot_a
```

I intentionally left:

```text
init_boot_b
```

stock.

This provides a useful untouched slot, but it also means an OTA may switch the watch to `_b` and root may disappear.

An OTA may also replace `init_boot`.

After a firmware update, **do not blindly flash your old rooted image**.

Instead:

```text
Check active slot
        ↓
Dump new stock init_boot
        ↓
Inspect new AVB configuration
        ↓
Patch new image with Magisk
        ↓
Re-sign for that firmware's AVB chain
        ↓
Verify
        ↓
Flash correct slot
```

---

# Things You Should NOT Do

- Do not flash someone else's `init_boot` without confirming the firmware is identical.
- Do not assume every S13/Core 5 has the same partition layout.
- Do not erase `init_boot` before making a verified backup.
- Do not flash the raw Magisk output without examining AVB.
- Do not disable AVB globally unless you actually understand why it is necessary.
- Do not immediately modify both A/B slots.
- Do not lose your original stock images.
- Do not assume a failed Fastboot dump created a valid image.

In particular, a failed:

```text
fastboot fetch init_boot_a
```

may leave behind a **zero-byte file**.

That file is not a firmware backup.

---

# Tested Configuration

This method successfully rooted:

```text
Model:          S13
Device:         dove
Product:        YBD_C29_OLED
Platform:       ASR8601
Android:        13
SDK:            33
Security patch: 2024-04-05

Build:
S13_C29_EN_V1.6_20251121

FOTA:
S13_C29_EN_V1.6_20251121_20251121-1207
```

Final state:

```text
Bootloader: LOCKED
AVB: ENABLED
Android: BOOTS
Magisk: RUNNING
Superuser: WORKING
Root UID: 0
```

No bootloader unlock was performed.

No global AVB disable was required.

Only `init_boot_a` was modified.

---

# Why This Works

The important part is the boot chain.

Conceptually, the tested firmware looks like:

```text
            Locked Bootloader
                   |
                   v
               vbmeta_a
                   |
        +----------+----------+
        |          |          |
        v          v          v
      boot     init_boot   vbmeta_system
                   |
                   v
           trusted RSA key
```

The stock `vbmeta_a` already trusted the key used to authenticate `init_boot`.

Therefore the process becomes:

```text
Stock init_boot
       |
       v
Magisk patches ramdisk
       |
       v
Generate correct AVB metadata
       |
       v
Sign with already-trusted key
       |
       v
Write through ASR8601 download mode
       |
       v
Locked bootloader verifies image
       |
       v
Android boots
       |
       v
Magisk starts
       |
       v
uid=0
```

So this was not really a traditional "bootloader bypass."

The more interesting issue was that this firmware's existing AVB configuration trusted a key for which signing material was available.

That allowed a modified `init_boot` to remain valid within the existing verified-boot chain.

---

# Recommended Backup

Keep something similar to:

```text
S13_ROOT_RECOVERY/
|
+-- stock/
|   +-- init_boot_a.img
|   +-- boot_a.img
|   +-- vendor_boot_a.img
|   +-- vbmeta_a.img
|   +-- vbmeta_system_a.img
|
+-- root/
|   +-- magisk_patched.img
|   +-- magisk_AVB_signed.img
|
+-- tools/
|   +-- ASRClientCore/
|   +-- ASR_USB_Drivers/
|   +-- platform-tools/
|   +-- Magisk.apk
|
+-- SHA256SUMS.txt
```

Keep at least one copy somewhere other than the computer used to modify the watch.

---

# Disclaimer

This is documentation of a procedure that worked on **my personally owned device**.

I cannot guarantee that another S13/Core 5 uses identical firmware, partition layouts, AVB keys, or ASR configuration.

Rooting and flashing firmware can make a device unbootable.

If you follow this guide, **dump and verify your own firmware first**.

---

# Credits

Thanks to:

- **YC-nw** for ASRClientCore  
  https://github.com/YC-nw/ASRClientCore

- **topjohnwu** and contributors for Magisk  
  https://github.com/topjohnwu/Magisk

- The Android Open Source Project for Android Verified Boot / `avbtool`  
  https://android.googlesource.com/platform/external/avb/

- **OpenAI ChatGPT** for extensive AI-assisted analysis, troubleshooting, reverse-engineering assistance, command generation, AVB analysis, and documentation during this project.

---

## Status

**Confirmed working on `S13_C29_EN_V1.6_20251121`.**

If you test this on another S13/Core 5 firmware revision, please open an issue with:

```text
ro.build.display.id
ro.build.fingerprint
ro.build.version.security_patch
ro.boot.slot_suffix
ro.product.device
ro.product.name
```

and, if possible, the output of `avbtool info_image` for your stock `init_boot` and `vbmeta`.****
