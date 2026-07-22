### TWRP device tree for Redmi Note 14 Pro (malachite)
=========================================

The Redmi Note 14 Pro (codenamed _"malachite"_) is a high-end, mid-range smartphone from Xiaomi.

It was released in September 26, 2024.

## Device specifications

Basic   |    Spec Sheet
-------:|:-------------------------
CPU    |  Octa-core (4x2.5 GHz Cortex-A78 & 4x2.0 GHz Cortex-A55)	
Chipset |  Mediatek Dimensity 7300-Ultra
GPU    |  Mali-G615 MC2
Memory |	8GB/12GB RAM (LPDDR5)	
Shipped Android Version | Android 14, up to HyperOS	
Storage |	256GB/512GB (UFS 3.1)	
Battery |	Non-removable Li-Po 5500 mAh battery	
Display |	1220 x 2712 pixels, 6.67 inches, 120 Hz, AMOLED

![Redmi Note 14 Pro](https://cdn.cnbj1.fds.api.mi-img.com/nr-pub/202409251505_ce7d2f6815bc93cd194fa6d320741795.png)

## Features

Works:

- [X] ADB
- [X] Decryption (Android 16)
- [X] Display
- [X] Fasbootd
- [X] Flashing
- [X] MTP
- [X] Sideload
- [X] USB OTG
- [X] Vibrator
- [X] Touch

## Compile

First checkout minimal twrp with aosp tree:

```
repo init --depth=1 -u https://github.com/minimal-manifest-twrp/platform_manifest_twrp_aosp.git -b twrp-14.1
repo sync -j$(nproc --all)
```

Then add these projects to .repo/manifest.xml:

```xml
<project path="device/xiaomi/malachite" name="mytiantian001/android_device_xiaomi_malachite_twrp" remote="github" revision="a16" />
```

Finally execute these:

```
source build/envsetup.sh
repopick <needed patch>
lunch twrp_malachite-eng
mka vendorbootimage -j$(nproc --all)
```
## To use it:

```
fastboot flash vendor_boot out/target/product/malachite/vendor_boot.img
```
