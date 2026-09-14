# Recovery startup and touch: September 2026 hardware test

This report covers two reproduced haptics failures and their fixes. It is not
a full recovery certification or a claim that the unmodified device tree boots
on every malachite firmware version.

## Test environment

- Device tree: `A16`, commit `c1d0d8a8ab0f385d23497dda3eb047f622291f6a`.
- TWRP core: TeamWin `android-14.1`, commit
  `426b747737e7ce9e9e17da5b4d2ba883f296aec7`.
- Physical malachite device identified by Android as model `24095PCADG`.
- Running Android kernel: `6.1.166-android14-11-g7d800f47b707`.
- The diagnostic build had crypto/FBE disabled after missing keystore build
  dependencies. This PR does not change the device tree's crypto settings.
- To obtain a working diagnostic boot, the test image used the platform
  ramdisk, modules, DTB and bootconfig from the working Lineage slot, a matching
  DTBO, and TWRP in a separate type-2 recovery fragment. These device-specific
  image adaptations are not included in this PR. Do not assume the repository's
  bundled hardware files were validated by this test.

## 1. Missing libraries prevented the recovery executable from starting

Starting `/system/bin/recovery` failed before its main function:

```text
CANNOT LINK EXECUTABLE "/system/bin/recovery": library "android.hardware.vibrator-V2-ndk.so" not found: needed by /system/lib64/libminuitwrp.so in namespace (default)
```

`libminuitwrp.so` also needed `android.hardware.vibrator-V2-cpp.so`. Both files
were built under `system/lib64`, but neither was included in the recovery
ramdisk. At the tested core revision, `prebuilt/Android.mk` included these
bindings inside the crypto/FBE block, while `TW_SUPPORT_INPUT_AIDL_HAPTICS`
caused libminuitwrp to link them independently of crypto/FBE.

The BoardConfig addition explicitly includes both libraries through
`TW_RECOVERY_ADDITIONAL_RELINK_LIBRARY_FILES`. It does not disable crypto or
add a different version of the libraries.

After copying those two built libraries into the running diagnostic recovery's
RAM filesystem, `/system/bin/linker64 --list /system/bin/recovery` succeeded.
The recovery process stayed running, reported its TWRP version, and the operator
confirmed the interface appeared on the screen.

## 2. Unlocking blocked the interface on an optional vibration

The interface then froze during the unlock slide. ADB remained available, and
the recovery process repeatedly logged:

```text
ServiceManagerCppClient: Waited for servicemanager.ready for a second, waiting another...
```

The slider called the vibration path on the interface thread. Its AIDL branch
called `AServiceManager_getService()`. The service manager was not running in
this diagnostic build, so the underlying `defaultServiceManager()` waited for
`servicemanager.ready` indefinitely.

The companion patch in `patches/` changes only the AIDL haptics path:

- Skip the optional vibration while `servicemanager.ready` is false.
- Once it is ready, use `AServiceManager_checkService()` to avoid retrying a
  missing vibrator service. A missing service remains a no-op.

This does not repair the service manager or vendor vibrator service themselves
and does not lazy-start an absent vibrator HAL. Vibration can be skipped; input
must remain usable. Other haptics backends are unchanged.

## Validation performed

1. Built the patched `libminuitwrp` with `mka libminuitwrp -j4` successfully.
2. Verified that its dynamic imports use `AServiceManager_checkService` instead
   of `AServiceManager_getService`.
3. Booted the diagnostic recovery on slot `b` and confirmed root ADB and the
   runtime slot before starting the interface.
4. Copied both missing AIDL libraries and the patched libminuitwrp into the
   RAM filesystem, verified their hashes, and checked the executable's library
   resolution before starting TWRP.
5. Asked the operator to retry unlocking, navigate the menus and return. The
   operator reported that it appeared to work well; the recovery log recorded
   navigation to Advanced and back to the main page.

The first attempt reproduced the linker failure; the second reproduced the
unlock freeze after adding only the missing libraries; the third ran with both
fixes and restored menu interaction. The final combined image was prepared and
checked offline, but its automatic startup after a fresh flash was not tested.

## Remaining validation

- A clean, complete build and fresh boot using this PR and the companion patch,
  with the intended firmware/kernel and matching hardware files.
- Decryption, `/data` access, backup/restore, flashing, sideload, fastbootd and
  USB OTG. The diagnostic build could not mount `/data`; it was not formatted.
- A working service manager and vendor haptics stack if vibration is required.

No full device logs, partition dumps, serial numbers or locally rebuilt images
are included in this contribution.
