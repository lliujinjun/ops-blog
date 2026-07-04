+++
date = '2026-07-04T20:04:00+08:00'
draft = false
title = '🖥️ Running CentOS 7 VM on CentOS Stream 9 with libvirt (Nested Virtualization)'
+++

## 📋 Overview

This post documents running a CentOS 7 VM inside a CentOS Stream 9 server using libvirt/KVM with nested virtualization. The VM is installed fully automatically via kickstart — no manual interaction needed.

**Host:** `remote-vm` — CentOS Stream 9, VMware VM, 3.5 GB RAM, 2 cores, AMD CPU with nested virtualization enabled.

---

## Prerequisites

### 1. Enable Nested Virtualization

If your host is inside a VMware VM, add this to the `.vmx` file on the VMware host:

```
vhv.enable = "TRUE"
```

Power off and power on the VM. Verify:

```bash
$ ls /dev/kvm
/dev/kvm

$ cat /sys/module/kvm_amd/parameters/nested
1
```

### 2. Install libvirt + KVM

```bash
sudo dnf install -y virt-install libvirt-daemon-kvm
sudo systemctl enable --now virtqemud.socket virtnetworkd.socket virtstoraged.socket virtnodedevd.socket
```

### 3. Create the default network

```bash
cat > /tmp/default-network.xml << "EOF"
<network>
  <name>default</name>
  <forward mode="nat"/>
  <bridge name="virbr0" stp="on" delay="0"/>
  <ip address="192.168.122.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.122.2" end="192.168.122.254"/>
    </dhcp>
  </ip>
</network>
EOF

sudo virsh net-define /tmp/default-network.xml
sudo virsh net-autostart default
sudo virsh net-start default
```

---

## Building a Custom Kickstart ISO

The most reliable way to do an automated CentOS 7 install is to bake the kickstart file directly into a custom ISO and modify the boot loader to use it automatically.

### Step 1: Extract the original ISO

```bash
mkdir -p /tmp/iso-orig /tmp/iso-custom
sudo mount -o loop /path/to/CentOS-7-x86_64-Minimal-2009.iso /tmp/iso-orig
cp -a /tmp/iso-orig/* /tmp/iso-custom/
cp -a /tmp/iso-orig/.??* /tmp/iso-custom/
sudo umount /tmp/iso-orig
```

### Step 2: Create the kickstart file

`/tmp/iso-custom/ks.cfg`:

```bash
install
cdrom
lang en_US.UTF-8
keyboard us
timezone Asia/Shanghai --isUtc
rootpw changeme
user --name=jellyfish --password=changeme
text
reboot
zerombr
clearpart --all --initlabel
autopart --type=lvm
network --bootproto=dhcp --device=link --onboot=yes --hostname=centos7-vm
%packages --ignoremissing
@core
openssh-server
sudo
%end
%post --log=/root/post-install.log
systemctl enable sshd
sed -i "s/#PermitRootLogin yes/PermitRootLogin yes/" /etc/ssh/sshd_config
sed -i "s/PasswordAuthentication no/PasswordAuthentication yes/" /etc/ssh/sshd_config
echo "jellyfish ALL=(ALL) NOPASSWD:***" > /etc/sudoers.d/jellyfish
%end
```

### Step 3: Replace isolinux config (skip graphical menu)

`/tmp/iso-custom/isolinux/isolinux.cfg`:

```bash
default linux
timeout 100
prompt 0
label linux
  kernel vmlinuz
  append initrd=initrd.img ks=cdrom:/ks.cfg console=tty0 console=ttyS0,115200n8 reboot=eject
```

The key points:
- `timeout 100` — auto-boot after 10 seconds (100 deciseconds)
- `prompt 0` — don't wait for user input
- `ks=cdrom:/ks.cfg` — point the installer to the kickstart file on the CDROM
- `console=ttyS0` — send output to serial console for headless access

### Step 4: Rebuild the ISO

```bash
cd /tmp/iso-custom
sudo genisoimage -o /var/lib/libvirt/images/centos7-auto.iso \
  -b isolinux/isolinux.bin -c isolinux/boot.cat \
  -no-emul-boot -boot-load-size 4 -boot-info-table \
  -R -J -V "CentOS 7 Auto" \
  .
```

---

## Launch the VM

```bash
sudo virt-install \
  --name centos7-vm \
  --memory 1024 \
  --vcpus 1 \
  --disk path=/var/lib/libvirt/images/centos7-vm.qcow2,size=10,format=qcow2 \
  --cdrom /var/lib/libvirt/images/centos7-auto.iso \
  --os-variant centos7.0 \
  --graphics none \
  --console pty,target_type=serial \
  --network network=default,model=virtio \
  --noautoconsole
```

The installation runs fully automated. To watch progress:

```bash
sudo virsh console centos7-vm --force
```

You'll see the kernel boot messages, package installation progress, and post-install scripts running in real time. Press **Ctrl+]** to disconnect from the console without stopping the VM.

### 🔍 Verifying the kickstart via console

When the installer finishes, the console will show:

```
Running post-installation scripts
...
Stopping Network Manager...
Deactivating swap...
Unmounting filesystems...
```

Followed by the VM shutting itself off (kickstart's `reboot` command triggers a clean shutdown in libvirt). This confirms everything worked.

After shutdown, start the VM:

```bash
sudo virsh start centos7-vm
```

---

## Post-Install Verification

After the VM reboots (kickstart's `reboot` command shuts it down), start it and verify:

```bash
$ sudo virsh start centos7-vm
Domain 'centos7-vm' started

$ sshpass -p changeme ssh root@192.168.122.16 "hostname && cat /etc/centos-release"
centos7-vm
CentOS Linux release 7.9.2009 (Core)
```

### Access from outside

```bash
# Via jump host
ssh -J jellyfish@remote-vm root@192.168.122.16
```

---

## 📊 Results

| Component | Status | Notes |
|---|---|---|
| Nested KVM in VMware | ✅ | AMD SVM, `/dev/kvm` |
| Custom kickstart ISO | ✅ | Auto-install, no interaction |
| CentOS 7 VM | ✅ | `centos7-vm`, 1 vCPU, 1 GB RAM, 10 GB disk |
| Network | ✅ | NAT via virbr0, DHCP at `192.168.122.16` |
| SSH access | ✅ | `root` / `changeme` |
| Systemd service | Optional | `virsh autostart centos7-vm` for auto-start |

---

## Alternative: Interactive Serial Console Install

If you prefer a manual install (or kickstart debugging fails), you can do the entire CentOS 7 installation interactively over a serial console. This uses the installer's text-mode TUI, which is fully functional over `ttyS0`.

### Create the VM with serial console kernel args

```bash
sudo virt-install \
  --name centos7-vm \
  --memory 1024 \
  --vcpus 1 \
  --disk path=/var/lib/libvirt/images/centos7-vm.qcow2,size=10,format=qcow2 \
  --location /var/lib/libvirt/images/CentOS-7-x86_64-Minimal-2009.iso \
  --extra-args="console=tty0 console=ttyS0" \
  --os-variant centos7.0 \
  --graphics none \
  --console pty,target_type=serial \
  --network network=default,model=virtio \
  --noautoconsole
```

> `--location` is required here instead of `--cdrom` because `--extra-args` only works with `--location` or `--kernel` installs.

### Connect and install

```bash
sudo virsh console centos7-vm --force
```

You'll see the boot menu, then the CentOS 7 text-based installer. Navigate using:
- **Tab** — move between fields
- **Space** — toggle checkboxes
- **Number keys** — select menu items
- **c** — continue / confirm
- **q** — go back

### Installer workflow

1. **Language** — default (English), press Enter
2. **Installation Summary** screen with these spokes to configure:

   | Spoke | Action |
   |---|---|
   | Time settings | Select your timezone |
   | Installation Destination | Select the disk, check "Automatically configure partitioning" |
   | Root password | Set a password |
   | User creation | Create a user (optional) |

3. After all required spokes show **[x]**, press **b** to begin installation
4. Wait for package install to complete (~5-10 minutes)
5. Press **Reboot** when prompted

After reboot, the VM will shut down (libvirt doesn't auto-restart). Start it manually:

```bash
sudo virsh start centos7-vm
```

Then SSH in:

```bash
ssh root@<vm-ip>
```

### Pros and cons vs kickstart

| Approach | Pros | Cons |
|---|---|---|
| **Interactive console** | Simple, no kickstart file needed, flexible | Requires manual input, can'\''t script |
| **Kickstart ISO** | Fully automated, reproducible | More setup, harder to debug |

---

## 💡 Lessons Learned

1. **📀 `--cdrom` + kickstart needs custom ISO** — You can't inject kernel args with `--cdrom` (only with `--location`). The most reliable approach is to bake the kickstart into a custom ISO and modify `isolinux.cfg` to use it.
2. **🖥️ Graphical menu blocks serial** — CentOS 7's default `vesamenu.c32` doesn't display on a serial console. Replace isolinux.cfg with a simple text config to see boot messages.
3. **⏰ `timeout` is in deciseconds** — `timeout 100` = 10 seconds. `timeout 600` = 60 seconds (the default timeout).
4. **🔌 VM reboots = shutdown in libvirt** — The kickstart `reboot` command shuts down the VM. You need to `virsh start` it again manually, or enable `virsh autostart`.
5. **🐌 Nested virt is slow but works** — 3.5 GB host RAM with a 1 GB VM leaves ~2.5 GB for the host. Installation completes in ~10 minutes (single vCPU).

---

*Running VMs inside VMs — it's VMs all the way down.* 🖥️
