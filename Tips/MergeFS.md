# MergeFS - Combining Multiple Storage Drives

## Summary
This guide is for people that have many external SSDs, USBs, HDDs or even internal drives that you want to merge and use as 1 filesystem. MergeFS allows you to combine multiple storage devices while also allowing you to unplug them and reconnect them without problems.

## Setup Process

1. **Plug in Drives**
   - Connect all storage devices you want to combine to your system

2. **Find Drive UUIDs**
   - Run `blkid` command to identify the unique identifiers of your drives

3. **Install mergerFS**
   - Install the mergerFS tool using your distribution's package manager (e.g., `apt install mergerfs`)

4. **Create Mount Points**
   - `mkdir -p /mnt/SSD, mkdir -p /mnt/USB, mkdir -p /mnt/merged`
   - Create mount points for each drive and a merged destination

5. **Mount First Drive**
   - `mount -U {UUID} /mnt/SSD`
   - Mount your first drive (SSD) using its UUID

6. **Mount Second Drive**
   - `mount -U {UUID} /mnt/USB`
   - Mount your second drive (USB) using its UUID

7. **Create the Merged Filesystem**
   - ```bash
     mergerfs -o defaults,allow_other,use_ino,direct_io,nonempty,cache.files=off,dropcacheonclose=true,symlinkify=true,category.create=mfs,fsname=virtualDisk /mnt/SSD:/mnt/USB /mnt/merged
     ```
   - Create a virtual merged filesystem that combines both drives with optimized settings to prevent data loss


## Reconnecting After Unplugging Drives

### Option 1: Only Remounting the Unplugged Drive

```bash
mount -U {UUID} /mnt/SSD  # or /mnt/USB
```
- Simply remount the drive that was unplugged using its UUID

### Option 2: If Option 1 Doesn't Work

If the first option doesn't work, try to remount both the unplugged drive and recreate the MergeFS filesystem:

1. **Remount the Unplugged Drive**
   ```bash
   mount -U {UUID} /mnt/SSD  # or /mnt/USB
   ```
   - Remount the drive that was unplugged using its UUID

2. **Recreate the Merged Filesystem**
   ```bash
   mergerfs -o defaults,allow_other,use_ino,direct_io,nonempty,cache.files=off,dropcacheonclose=true,symlinkify=true,category.create=mfs,fsname=virtualDisk /mnt/SSD:/mnt/USB /mnt/merged
   ```
   - Recreate the merged filesystem with all drives 