Rooting a Pixel 6 Pro
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Initial Root and OTA Update Procedure for Google Pixel 6 Pro
running Android 12.

At least adb 31.0.3 is needed ie via google sdk install.
Instructions were tested on an Ubuntu 22.04 host.

.. contents::


Host
----

Preparing the host

- check android sdk version ``adb --version``
- compare to https://developer.android.com/studio/releases/platform-tools
- if needed, download, unzip android sdk platform tools
- alternate: install android-sdk-platform-tools-common
- add sdk bindir to executable path
- add self to plugdev group and newgrp (ubuntu)
- ``adb reboot-bootloader``


Device
------

Preparing the device

- start
- configure wifi
- disable system updates
- about -> build number -> developer
- developer options
- unlock bootloader
- authorize usb debugging
- install or update magisk canary >= 24308 with 4cff0384 patch (see git log)
- rebuild magisk if necessary, sometimes patch isn't released as canary yet
- last tested on magisk 9183a0a6 ie 25101 ie v25.1
- reboot if it asks


Backup
------

- install or update swift backup
- run device + cloud backups
- perms check: ``adb shell``; ``su``; ``cd /sdcard``; ``find -not -perm -040``
- remove any old backups no longer needed to free up space
- ``adb pull /sdcard``


Mkboot
------

Creating the patched Magisk from the factory boot.img

- download and unzip factory.zip
- unzip extracted device-build.zip
- ``adb push boot.img /sdcard/boot.img`` # warning: /sdcard/ is bad
- patch boot image with magisk
- adb pull patched boot image


Init
----

Initialize the device to stock at any factory release, and then
root it with Magisk.  This is when first getting device.  Remove
``-w`` from the update command to not wipe.

- ``adb reboot bootloader``
- ``fastboot flash bootloader bootloader*.img``
- ``fastboot reboot-bootloader``
- ``fastboot flash radio radio*.img``
- ``fastboot reboot-bootloader``
- ``fastboot --disable-verity --disable-verification --skip-reboot -w update image*.zip``
- voldown -> reboot to bootloader # important: wait for flash to finish
- ``fastboot --slot=all flash boot boot-patched.img``
- ``fastboot reboot``
- optionally copy apps and data from old phone
- repeat `device`_
- confirm su -> root
- allow com.android.shell for adb su (TODO revoke)


Update
------

Apply OTA and reclaim root

- extract newest factory unzip and its nested image.zip
- extract boot.img, vbmeta.img
- download any intermediary, and newest ota zip, no need to unzip
- patch boot image following `mkboot`_ and adb pull it back
- adb reboot recovery -> "no command"
- hold power + once volup -> release volup -> release power -> recovery
- apply update from adb
- ``adb sideload ota.zip`` # do not reboot
- repeat in sequence for any other otas # todo: can skip to latest one?
- recovery -> reboot to bootloader
- ``fastboot --disable-verity --disable-verification --slot=all flash vbmeta vbmeta.img``
- ``fastboot --slot=all flash boot boot-patched.img``
- recovery -> start

This sequence avoids any boot without root.  If there are boot
issues encountered, flash vendor boot.img, boot without root,
and try to make a new patched boot image from the new
[now-ota-updated] Android and re-flash the boot partition.
Early in Pixel 6 series, there was a time when Magisk needed to
patch the boot image on the actual end-host that it would boot
into, or the patched boot wouldn't work properly; this
necessitated a second boot during update (first into non-root).
However, this doesn't seem to be an issue any longer at this
time, so patched boot can be flashed right after vbmeta, prior
to first boot.

For that matter, the whole update can be done within the Magisk
app now also, by uninstalling Magisk (within Magisk), taking the
OTA as a Software Update, and then re-installing Magisk to the
inactive slot.  This method broke for a long time, but is now
fixed.
