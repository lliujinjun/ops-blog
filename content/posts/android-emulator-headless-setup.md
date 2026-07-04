+++
date = '2026-07-04T17:40:00+08:00'
draft = false
title = '🤖 Android Emulator on CentOS Stream 9 (Headless, KVM, Systemd)'
+++

## 📋 Overview

This post documents setting up the Android SDK + AVD emulator on a CentOS Stream 9 server running inside a **VMware VM** with **nested virtualization**. The emulator runs headless (no display), auto-starts via systemd, and is reachable via ADB.

**Server:** `remote-vm` — CentOS Stream 9, VMware VM, AMD CPU, 3.5 GB RAM, 2 cores

---

## Prerequisites

### 1. Enable Nested Virtualization

Since the server is a VMware VM, nested virtualization needs to be enabled in the VM's `.vmx` file:

```
vhv.enable = "TRUE"
```

After adding this, power off and power back on the VM (no soft reboot). Verify:

```bash
$ ls -l /dev/kvm
crw-rw-rw-. 1 root kvm 10, 232 Jul  4 17:21 /dev/kvm

$ grep -E "vmx|svm" /proc/cpuinfo | head -1
flags: ... svm ... npt ... (AMD) or vmx ... (Intel)

$ cat /sys/module/kvm_amd/parameters/nested  # or kvm_intel
1
```

### 2. Disable SELinux

The emulator binaries crashed with SEGV under SELinux Enforcing due to `user_home_t` context. Setting SELinux to Permissive resolves this:

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

A reboot is required for full disable — for testing, `setenforce 0` is sufficient.

### 3. Install Dependencies

```bash
sudo dnf install -y java-21-openjdk-headless wget unzip glibc-devel zlib-devel \
  pulseaudio-libs libX11 libxkbfile libXcomposite libXcursor libXdamage \
  libXfixes libXi libXrandr libXrender libXtst libXScrnSaver libpng
```

Java 21 is required by the Android SDK tools. The X11 and PulseAudio libs are needed by the emulator's QEMU binary (even in headless mode).

---

## Step 1: Install the Android SDK

```bash
# Download command-line tools
curl -sL -o /tmp/cmdline-tools.zip \
  "https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip"

# Set up proper SDK directory structure
# Android expects: <sdk>/cmdline-tools/latest/bin/sdkmanager
mkdir -p /opt/android-sdk/cmdline-tools/latest
cd /opt/android-sdk/cmdline-tools/latest
unzip -q /tmp/cmdline-tools.zip
# The zip contains cmdline-tools/ - move contents up
mv cmdline-tools/* . 2>/dev/null; mv cmdline-tools/.* . 2>/dev/null; rmdir cmdline-tools

# Set up environment
export ANDROID_HOME=/opt/android-sdk
export PATH=$ANDROID_HOME/cmdline-tools/latest/bin:$PATH

# Accept licenses
yes | sdkmanager --licenses
```

> **Why `/opt/android-sdk`?** systemd services can't exec files inside `/home` under SELinux. Putting the SDK in `/opt` avoids this problem entirely.

---

## Step 2: Install Platform, System Image, and Emulator

```bash
sdkmanager "platform-tools" "platforms;android-34"
sdkmanager "system-images;android-34;google_apis;x86_64"
sdkmanager "emulator"
```

Total download: **~6.6 GB** including:
- Android SDK Platform 34 (~200 MB)
- Platform tools (adb, fastboot) (~100 MB)
- Emulator binary and libraries (~1 GB)
- System image (Google APIs x86_64, ~2 GB zipped)

---

## Step 3: Create an AVD

```bash
export PATH=$ANDROID_HOME/platform-tools:$ANDROID_HOME/emulator:$PATH

echo no | avdmanager create avd \
  -n test-device \
  -k "system-images;android-34;google_apis;x86_64" \
  --device pixel_6 --force
```

Adjust the RAM to fit your server resources. For a 3.5 GB server, limit to 1024 MB:

```bash
sed -i "s/hw.ramSize=.*/hw.ramSize=1024/" ~/.android/avd/test-device.avd/config.ini
```

---

## Step 4: Systemd Service (Persistent Emulator)

The emulator needs to stay alive across SSH disconnects. A systemd service handles this:

```ini
[Unit]
Description=Android Emulator (test-device)
After=network.target

[Service]
User=jellyfish
Type=simple
ExecStart=/bin/bash /opt/android-sdk/emulator/launcher.sh \
  -avd test-device -no-window -no-audio -no-snapshot \
  -no-boot-anim -no-metrics -gpu guest -memory 1024
ExecStop=/bin/bash -c "/opt/android-sdk/platform-tools/adb emu kill"
Restart=on-failure
RestartSec=30
RestartSteps=3

[Install]
WantedBy=multi-user.target
```

The `launcher.sh` wrapper sets the environment variables that the emulator's bundled libraries need:

```bash
#!/bin/bash
export ANDROID_HOME=/opt/android-sdk
export LD_LIBRARY_PATH=$ANDROID_HOME/emulator/lib64:$ANDROID_HOME/emulator/lib
export PATH=$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$ANDROID_HOME/emulator:/usr/local/bin:/usr/bin
exec $ANDROID_HOME/emulator/emulator "$@"
```

Enable and start:

```bash
sudo systemctl enable android-emulator
sudo systemctl start android-emulator
```

---

## Step 5: Verify

Wait 2-3 minutes for the first boot (Android optimizes app cache on first boot):

```bash
$ export ANDROID_HOME=/opt/android-sdk
$ export PATH=$ANDROID_HOME/platform-tools:$PATH

$ adb devices
List of devices attached
emulator-5554	device

$ adb wait-for-device
$ adb shell getprop sys.boot_completed
1

$ adb shell getprop ro.build.version.sdk
34

$ adb shell getprop ro.build.version.release
14
```

---

### ⚙️ Quick Test

```bash
adb shell "echo hello from android && uname -a"
Linux localhost 5.15.155-android14-11-g5b0ba41eebbf #1 SMP PREEMPT Wed ...
```

---

## 📊 Results

| Component | Status | Notes |
|---|---|---|
| KVM / nested virt | ✅ | AMD SVM, `/dev/kvm` |
| SDK + platform 34 | ✅ | `/opt/android-sdk` (6.6 GB) |
| Emulator binary | ✅ | v36.6.11, SwiftShader guest GPU |
| AVD `test-device` | ✅ | Android 14, x86_64, 1024 MB RAM |
| Systemd service | ✅ | Auto-start, survives reboot |
| ADB connectivity | ✅ | `emulator-5554 device` |

---

## 💡 Lessons Learned

1. **🏠 systemd can't exec from `/home`** — Files in user home directories fail with `EXEC` (exit code 203) under SELinux. Move the SDK to `/opt/` and restorecon it.
2. **🔒 SELinux blocks everything** — The emulator consistently crashed with SEGV during Vulkan initialization under SELinux Enforcing. Setting to Permissive resolved it.
3. **🖥️ GPU mode matters headless** — `-gpu guest` (guest-side rendering) works reliably without a physical GPU. `-gpu swiftshader_indirect` and `auto` both crashed.
4. **🐏 RAM is tight** — 3.5 GB total means limiting the emulator to 1024 MB. The emulator uses ~300-500 MB at runtime, leaving room for other services.
5. **⏱️ First boot is slow** — Android 14 takes 2-3 minutes on first boot (cache optimization). Subsequent boots are faster with `-no-snapshot`.

---

*An Android emulator running in a VMware VM on a headless server. Not something you see every day.* 🤖
