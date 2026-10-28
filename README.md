# Cybersecurity Testing Lab: VirtualBox + Kali Linux

**Task:** WK1-PM1, Lab Setup (VirtualBox and Kali Linux)
**Programme:** Networkwalks Academy, Cybersecurity Internship
**Author:** Takudzwanashe Musere
**Date completed:** October 2026

This repository documents how I built an isolated cybersecurity practice lab on my own laptop. Kali Linux runs as the attacking machine inside VirtualBox, on a private NAT Network (`10.0.0.0/24`). It also records the problems I hit along the way and how I fixed them.

> The lab is for learning and authorised practice only. Only test systems you own or have explicit permission to test.

---

## 1. Task requirements and evidence

| # | Requirement | Status | Evidence |
|---|---|---|---|
| 1 | Use VirtualBox as the base (latest recommended version) | Done (7.2.22) | `screenshots/01-virtualbox-installed.png` |
| 2 | Set up Kali Linux as the attacking machine | Done (2026.2) | `screenshots/02-kali-desktop.png` |
| 3 | Network in subnet `10.0.0.0/24` | Done | `screenshots/03-natnetwork.png` |
| 4 | Use a NATNetwork with `10.0.0.0/24` | Done | `screenshots/03-natnetwork.png` |
| 5 | Clipboard and drag-and-drop enabled | Done (Bidirectional) | `screenshots/04-clipboard-dragdrop.png` |
| 6 | Shared folder with host `Downloads` | Done | `screenshots/05-shared-folder.png` |
| 7 | Kali IP address `10.0.0.2/24` | Done | `screenshots/06-ip-address.png` |
| 8 | Kali has full internet access | Done | `screenshots/07-internet-test.png` |
| 9 | Snapshot of the VM | Done | `screenshots/08-snapshot.png` |

---

## 2. Lab architecture

```
Windows 10 host (Lenovo ThinkPad T440)
└── Oracle VirtualBox 7.2.22
    └── NatNetwork: 10.0.0.0/24 (DHCP enabled, gateway 10.0.0.1)
        └── Kali Linux 2026.2 (attacker): 10.0.0.2/24
            (space reserved for optional future VMs: Windows 11/10/7, Android)
```

---

## 3. Environment

| Item | Detail |
|---|---|
| Host OS | Windows 10 Pro |
| Host hardware | Intel Core i5-4300U, 8 GB RAM |
| Hypervisor | Oracle VirtualBox 7.2.22 |
| Guest OS | Kali Linux 2026.2 (pre-built VirtualBox image, amd64) |
| VM resources | 2 CPUs, 2048 MB RAM, VMSVGA graphics, 128 MB video memory |
| VM storage | Kept on a separate drive (`D:\VirtualBox VMs`) because the system drive was nearly full |

---

## 4. Setup steps

### Step 0: Enable hardware virtualization (BIOS)

VirtualBox reported `VERR_VMX_MSR_ALL_VMX_DISABLED` when I first started Kali, which means VT-x was off in the BIOS.

1. Restart and press `Enter`, then `F1`, at the Lenovo logo to open BIOS Setup.
2. Go to **Security → Virtualization → Intel Virtualization Technology** and set it to **Enabled**.
3. Press `F10` to save and exit.
4. Confirm in **Task Manager → Performance → CPU** that *Virtualization: Enabled*.

### Step 1: Install 7-Zip and VirtualBox

1. Install 7-Zip from <https://7-zip.org/download.html> (needed to extract the `.7z` Kali image).
2. Download VirtualBox from <https://www.virtualbox.org/wiki/Downloads> and install it.
3. Set **File → Preferences → General → Default Machine Folder** to a folder on the larger drive.

### Step 2: Create the NAT Network

1. In VirtualBox go to **File → Tools → Network → NAT Networks**.
2. Click **Create** and set:
   - **Name:** `NatNetwork`
   - **IPv4 Prefix:** `10.0.0.0/24`
   - **Enable DHCP:** ticked
   - **Enable IPv6:** unticked

### Step 3: Download and import Kali Linux

1. Download the **VirtualBox** pre-built image from <https://www.kali.org/get-kali/> (Virtual Machines section).
2. Extract the `.7z` file with 7-Zip into the VM folder.
3. Double-click the `.vbox` file (or **Machine → Open**) to register the VM.

### Step 4: Configure the VM settings

With the VM powered off, open **Settings**:

| Section | Setting |
|---|---|
| System | Base memory 2048 MB, 2 processors |
| Display | Graphics controller VMSVGA, 128 MB |
| Network → Adapter 1 | Attached to **NAT Network**, Name **NatNetwork**, Promiscuous Mode **Allow All** |
| General → Features | Shared Clipboard **Bidirectional**, Drag-and-Drop **Bidirectional** |

### Step 5: Install the VirtualBox guest tools in Kali

The guest tools make the clipboard, drag-and-drop and shared folders work.

```bash
sudo apt update
sudo apt install -y virtualbox-guest-x11
sudo usermod -aG vboxsf kali
sudo reboot
```

### Step 6: Add the shared folder

1. With Kali running, open **Devices → Shared Folders → Shared Folders Settings...**
2. Click **Add** and set:
   - **Folder Path:** the Windows `Downloads` folder
   - **Folder Name:** `Downloads`
   - **Read-only:** unticked (full access)
   - **Auto-mount:** ticked
   - **Make Permanent:** ticked
3. In Kali, the folder appears on the desktop as `sf_Downloads` and under `/media`. If it does not mount automatically:

```bash
sudo mkdir -p /mnt/downloads
sudo mount -t vboxsf Downloads /mnt/downloads
ls /mnt/downloads
```

### Step 7: Set the static IP address

In Kali, open **Network icon → Edit Connections → Wired connection 1 → IPv4 Settings**:

| Field | Value |
|---|---|
| Method | Manual |
| Address | `10.0.0.2` |
| Netmask | `24` |
| Gateway | `10.0.0.1` |
| DNS servers | `8.8.8.8` (use `10.0.0.1` if internet gives issues) |

Then apply the change with the fix recommended for newer VirtualBox 7 / Kali 2026.1+ builds:

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.dad-timeout 0
sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```

### Step 8: Take a snapshot

1. Shut Kali down cleanly: `sudo poweroff`.
2. In VirtualBox Manager select the VM, open the **Snapshots** tab and click **Take**.
3. Name it `Lab setup complete (10.0.0.2)` and add a short description.

---

## 5. Verification

Run these inside Kali:

```bash
ip a                      # eth0 should show inet 10.0.0.2/24
ping -c 3 10.0.0.1        # NAT Network gateway reachable
ping -c 3 8.8.8.8         # internet reachable by IP
ping -c 3 kali.org        # DNS resolution and internet working
ls /mnt/downloads         # host Downloads folder visible (or ls /media/sf_Downloads)
```

Expected results:

- `eth0` is `UP` with `10.0.0.2/24`.
- All pings return replies with 0% packet loss.
- The shared folder lists files from the Windows `Downloads` directory.
- Text copied on Windows pastes into Kali (`Ctrl + Shift + V` in the terminal) and the other way round.

---

## 6. Problems I ran into and how I fixed them

| Problem | Cause | Fix |
|---|---|---|
| `VERR_VMX_MSR_ALL_VMX_DISABLED` / "VT-x is disabled in the BIOS" | Virtualization was off in BIOS | Enabled Intel VT-x in BIOS (Step 0) |
| System drive almost full (under 1 GB free) | The 15 GB Kali disk would not fit on C: | Moved the VM and set the default machine folder to the D: drive |
| `apt update` failed with "Not live until ..." signature errors | The VM clock was wrong after an abrupt shutdown | `sudo timedatectl set-timezone Africa/Harare` and `sudo timedatectl set-ntp true` |
| `apt full-upgrade` failed with "Unable to connect" to the mirrors | The VM was on plain NAT and the connection was unstable | Switched to the NAT Network, applied the `nmcli` fix, then retried |
| VM would not accept setting changes and was marked *Saved* | It had been suspended instead of shut down | Resumed it, ran `sudo poweroff`, then changed the settings |
| `/media` empty after adding the shared folder | The folder had been added while the VM was off and Auto-mount was not set | Re-added it from **Devices → Shared Folders** while the VM was running with Auto-mount ticked |

---

## 7. Repository layout

```
.
├── README.md
└── screenshots/
    ├── 01-virtualbox-installed.png
    ├── 02-kali-desktop.png
    ├── 03-natnetwork.png
    ├── 04-clipboard-dragdrop.png
    ├── 05-shared-folder.png
    ├── 06-ip-address.png
    ├── 07-internet-test.png
    └── 08-snapshot.png
```

---

## 8. Next steps (optional phase)

Step 7 of the task suggests installing additional VMs (Windows 11/10/7 or Android) on the same NAT Network, ping-testing between machines, and snapshotting each one. That phase is not part of this submission.

---

## Disclaimer

This lab exists for education and authorised security testing only. The author is not responsible for misuse of the tools or techniques used in it.
