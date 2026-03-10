# System Drive Migration: HDD → SSD via Live USB

A step-by-step guide to cloning a Proxmox installation from an old HDD to a new SSD using a Ubuntu Live USB. No reinstallation required — everything including LVM, LXCs, and bootloader is preserved.

---

## Example Setup

> Replace these with your own hardware details.

| Component | Details |
|---|---|
| **Machine** | Any mini PC or desktop with a 2.5" SATA bay |
| **Source Drive** | `[YOUR SOURCE DRIVE SIZE & MODEL]` (HDD) |
| **Target Drive** | `[YOUR TARGET DRIVE SIZE & MODEL]` (SSD) |
| **OS** | Proxmox VE |

---

## Prerequisites

- A Ubuntu Live USB (or any Linux Live USB)
- A SATA-to-USB adapter to connect the new drive externally
- Both drives available during the cloning process
- Access to your machine's boot menu (commonly **F12**, **F11**, or **ESC** depending on manufacturer)

---

## Step 1: Connect the New Drive

1. Power off the machine completely and unplug power
2. Connect the new SSD via a SATA-to-USB adapter
3. Power the machine back on

---

## Step 2: Boot into Ubuntu Live USB

1. Plug the Live USB into the machine
2. Power on and press your boot menu key (**F12**, **F11**, or **ESC** — varies by manufacturer)
3. Select your USB drive from the boot menu
4. Choose **"Try Ubuntu"** — do NOT select Install

---

## Step 3: Identify Your Drives

Open a terminal and run:

```bash
lsblk
```

You should see something like:

```
NAME   SIZE  TYPE MOUNTPOINTS
sda    [X]G  disk             ← Source (old drive with Proxmox)
sdb    [Y]G  disk             ← Target (new drive, empty)
```

> ⚠️ **Critical:** Double check which is source and which is target before proceeding. Writing to the wrong drive will destroy your data.

---

## Step 4: Clone the Drive

Run the following `dd` command to do a full sector-by-sector clone:

```bash
sudo dd if=/dev/sda of=/dev/sdb bs=4M status=progress conv=noerror,sync
```

| Flag | Meaning |
|---|---|
| `if=/dev/sda` | Input (source — old HDD) |
| `of=/dev/sdb` | Output (target — new SSD) |
| `bs=4M` | Block size for efficient transfer |
| `status=progress` | Shows live progress |
| `conv=noerror,sync` | Continues on errors, pads bad blocks |

This copies everything — bootloader, partition table, LVM volumes, and all LXC data.

> ⏱️ **Expected time:** 30–60 minutes depending on drive size and USB speed.

Wait until you see the completion message:

```
XXX bytes copied, XXX seconds, XXX MB/s
```

---

## Step 5: Swap the Drives

1. Shut down the Live USB session cleanly:
```bash
sudo shutdown now
```
2. Unplug the Live USB
3. Remove the old HDD from the machine
4. Insert the new SSD into the drive bay
5. Keep the old HDD somewhere safe as a backup

---

## Step 6: Boot Proxmox

1. Power on the machine normally (no USB drives attached)
2. Proxmox should boot from the new SSD automatically
3. Verify everything works — check your LXCs, web UI, etc.

---

## Notes

- The new drive will have unallocated space after cloning since the partition layout is copied from the smaller source drive. See the [Proxmox LVM Expansion guide](#) to reclaim the remaining space.
- Keep the old HDD for 2–3 weeks before wiping it, just in case anything unexpected comes up.
- If Proxmox doesn't boot, re-enter the boot menu and manually select the new drive.

---

## Why This Works

`dd` performs a bit-for-bit copy of the entire disk, meaning it preserves:

- ✅ GRUB bootloader
- ✅ Partition table (GPT/MBR)
- ✅ EFI partition
- ✅ LVM physical volumes, volume groups, and logical volumes
- ✅ All LXC/VM data
- ✅ Proxmox configuration

No reinstallation or reconfiguration needed.
