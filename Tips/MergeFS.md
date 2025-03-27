# MergeFS - Combining Multiple Storage Drives

## Setup Process

1. Plug in Drives
   - Connect all storage devices you want to combine to your system

2. blkid to find the UUID of the drives
   - Run `blkid` command to identify the unique identifiers of your drives

3. Install mergerFS
   - Install the mergerFS tool using your distribution's package manager (e.g., `apt install mergerfs`)

4. mkdir -p /mnt/SSD, mkdir -p /mnt/USB, mkdir -p /mnt/merged
   - Create mount points for each drive and a merged destination

5. mount -U {UUID} /mnt/SSD
   - Mount your first drive (SSD) using its UUID

6. mount -U {UUID} /mnt/USB
   - Mount your second drive (USB) using its UUID

7. mergerfs -o defaults,allow_other,use_ino,direct_io,nonempty,cache.files=off,dropcacheonclose=true,symlinkify=true,category.create=mfs,fsname=virtualDisk /mnt/SSD:/mnt/USB /mnt/merged
   - Create a virtual merged filesystem that combines both drives with optimized settings to try and prevent data loss


## When wanting to UNPLUG do the following to RECONNECT:

### Option 1 (Only remounting the unplugged drive): 

a. mount -U {UUID} /mnt/SSD or /mnt/USB
   - Simply remount the drive that was unplugged using its UUID

### Option 2 (If Option 1 doesn't work, try to remount the unplugged drive and the MergerFS filesystem as well):

b. mount -U {UUID} /mnt/SSD or /mnt/USB
   - Remount the drive that was unplugged using its UUID

c. mergerfs -o defaults,allow_other,use_ino,direct_io,nonempty,cache.files=off,dropcacheonclose=true,symlinkify=true,category.create=mfs,fsname=virtualDisk /mnt/SSD:/mnt/USB /mnt/merged
   - Recreate the merged filesystem with all drives 