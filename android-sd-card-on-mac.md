# Accessing an Android SD Card from macOS

macOS does not natively mount Android storage in Finder because Android normally exposes it using **MTP (Media Transfer Protocol)** rather than as a conventional USB disk.

This guide covers the most useful options, with **ADB** recommended for reliable command-line transfers.

## Options at a glance

| Requirement | Recommended option |
| --- | --- |
| One-off or very large transfer | Remove the SD card and use a USB card reader |
| Reliable command-line copying | ADB |
| Browse the phone in Finder | MacDroid |
| Free graphical transfers | OpenMTP |
| A genuinely mounted remote directory | Termux and SSHFS |

## Option 1: Use a card reader

If the microSD card is removable, turn off the phone, remove the card and connect it to the Mac with a USB or USB-C card reader. It should appear under **Locations** in Finder.

If it does not appear, open **Disk Utility** and select **View → Show All Devices**. Do not initialise or erase the card if it contains files you need.

This is normally the fastest and most reliable method. It will not work if Android formatted the card as encrypted **internal/adopted storage**; such a card must be accessed through the original phone.

## Option 2: Transfer files with ADB

ADB is reliable and scriptable, although it does not mount the card in Finder.

### Install ADB

Using Homebrew:

```bash
brew install --cask android-platform-tools
```

Confirm the installation:

```bash
command -v adb
adb version
```

### Enable USB debugging

On the Android phone:

1. Open **Settings → About phone**.
2. Tap **Build number** seven times to enable Developer options.
3. Open **Settings → Developer options**.
4. Enable **USB debugging**.
5. Connect the phone using a USB data cable.
6. Unlock the phone and choose **File transfer / Android Auto** under USB preferences.
7. Approve the **Allow USB debugging?** prompt and optionally select **Always allow from this computer** for your own Mac.

Check the connection:

```bash
adb devices -l
```

A working connection resembles:

```text
R58XXXXXXXX device product:example model:example device:example
```

If it says `unauthorized`, unlock the phone and approve the debugging prompt.

### Locate the SD card

List Android storage:

```bash
adb shell ls -la /storage
adb shell sm list-volumes all
```

The removable card will usually have an identifier resembling:

```text
1234-5678
```

Its path is therefore normally:

```text
/storage/1234-5678
```

Browse it:

```bash
adb shell ls -la "/storage/1234-5678"
```

Replace `1234-5678` in subsequent examples with the identifier shown by your phone.

### Copy the complete SD card to the Mac

Create a destination:

```bash
mkdir -p "$HOME/Android-SD-Backup"
```

Copy the card:

```bash
adb pull "/storage/1234-5678/." \
  "$HOME/Android-SD-Backup/"
```

Keep the phone connected and unlocked during a large transfer.

### Copy files back to the SD card

For example, copy a local file into the card's Download directory:

```bash
adb push "/path/to/file" \
  "/storage/1234-5678/Download/"
```

## Troubleshooting an empty `adb devices` result

If the command shows only `List of devices attached`, work through these checks.

### Confirm macOS sees the USB device

```bash
system_profiler SPUSBDataType | \
  grep -i -A 12 -E 'android|samsung|google|pixel|motorola|xiaomi'
```

If the phone is absent, try:

- A known data-capable USB cable; many cables provide power only.
- A different USB port.
- Connecting directly rather than through a hub.
- Unlocking the phone and selecting **File transfer / Android Auto**.

### Restart ADB

Quit OpenMTP or MacDroid first, then run:

```bash
adb kill-server
adb start-server
adb devices -l
```

### Reset USB-debugging authorisation

On Android:

1. Open **Developer options**.
2. Select **Revoke USB debugging authorisations**.
3. Turn USB debugging off and back on.
4. Disconnect and reconnect the phone.
5. Approve the new authorisation prompt.

Then retry:

```bash
adb devices -l
```

### Use wireless ADB instead

Android 11 and later can connect without USB. The Mac and phone must be on the same Wi-Fi network.

On the phone, enable **Developer options → Wireless debugging**, select **Pair device with pairing code**, and note the displayed address and port.

Pair from the Mac:

```bash
adb pair PHONE_IP:PAIRING_PORT
```

Enter the six-digit pairing code. Then connect using the separate debugging port shown on the main Wireless debugging screen:

```bash
adb connect PHONE_IP:DEBUGGING_PORT
adb devices -l
```

## Option 3: Browse the phone with MacDroid

MacDroid exposes Android internal storage and SD cards through Finder. It supports USB MTP, ADB and wireless connections.

The free edition is suitable when copying from Android to Mac. Paid functionality is most useful for regular two-way transfers, editing, renaming, deleting and organising Android files from Finder.

MacDroid is most likely worth paying for when Finder integration is used frequently. For occasional backups, ADB or a card reader is usually sufficient. Use the trial with the actual phone and test large folders, files over 4 GB, reconnecting and transfers in both directions before purchasing.

## Option 4: Use OpenMTP

OpenMTP is a free, open-source graphical MTP file manager. Connect the unlocked phone, select **File transfer**, open OpenMTP and switch from internal storage to the SD card.

It is useful for occasional drag-and-drop transfers, but the storage appears inside OpenMTP rather than as a normal Finder volume.

## Option 5: Mount through Termux and SSHFS

For a genuine mounted directory, Android can run an SSH server through Termux and the Mac can mount the SD card over Wi-Fi using SSHFS.

In Termux:

```bash
termux-setup-storage
pkg update
pkg install openssh
passwd
sshd
```

Termux normally listens on port `8022`. Obtain the username and phone address:

```bash
whoami
ip addr
```

The SD card is commonly exposed in Termux as:

```text
~/storage/external-1
```

After installing macFUSE and a compatible SSHFS implementation on the Mac, create a mount point:

```bash
mkdir -p "$HOME/Android-SD"
```

Mount it, replacing the username and IP address:

```bash
sshfs -p 8022 \
  TERMUX_USERNAME@PHONE_IP:/data/data/com.termux/files/home/storage/external-1 \
  "$HOME/Android-SD"
```

Open it in Finder:

```bash
open "$HOME/Android-SD"
```

Unmount it when finished:

```bash
umount "$HOME/Android-SD"
```

The phone and Mac must remain on the same network, and Termux's SSH server must keep running.

## Security after using ADB

USB debugging gives an authorised computer substantial access to the phone. When finished with a one-off transfer:

1. Disable **USB debugging** and **Wireless debugging**.
2. Use **Revoke USB debugging authorisations** if the Mac should not remain trusted.
3. Stop Termux's SSH server, if used:

```bash
pkill sshd
```

For routine use with your own trusted Mac, leaving its ADB authorisation registered is convenient, but USB debugging should still be disabled when it is not needed.
