# shrink-backup is a bash script for backing up your SBC:s `/dev/mmcblk0` (SD-card) into an img file

_I made this script because I wanted a universal method of backing up my SBC:s into img files as fast as possible (with rsync), no matter what os is in use._

[shrink-backup](shrink-backup)

Tested on **Raspberry Pi** os, **Armian** and **Manjaro-arm** with ext4 root partition.
Autoexpansion will ONLY work on these three at the moment.
If none of the above os's are detected, you will be informed that the autoexpand functionality is not available but will not fail the backup.

**Do not forget to make the script executable after downloading it.**

## Usage:
```
sudo shrink-backup -h
Script for creating an .img file and subsequently keeing it updated (-B), autoexpansion is enabled by default
Directory where .img file is created is automatically excluded in backup
########################################################################
Usage: sudo shrink-backup [-Uatyedh] imagefile.img [extra space (MB)]
  -U         Update the img file (rsync to existing backup .img), no resizing, -a is ignored
  -a         Let resize2fs decide minimum space (extra space is ignored), disabled if using -U
  -t         Use exclude.txt in same folder as script to set excluded directories
             One directory per line: "/dir" or "/dir/*" to only exclude contents
  -y         Disable prompts in script
  -e         DO NOT expand filesystem when image is booted
  -d         Write debug messages in log file shrink-backup.log in same directory as script
  -h --help  Show this help snippet
########################################################################
Example: sudo shrink-backup -at /path/to/backup.img
Example: sudo shrink-backup -e -y /path/to/backup.img 1000
Example: sudo shrink-backup -Ut /path/to/backup.img
```

The folder where the img file is created will ALWAYS be excluded in the backup.
If `-f` option is selected, exclude.txt **MUST exist** (but can be empty) within the **directory where the script is located** or the script will exit with an error.

Use one directory per line in exclude.txt.
`/directory/*` = create directory but exclude content.
`/directory` = exclude the directory completely.

If `-f` is **NOT** selected the following folders will be excluded:
```
/lost+found
/proc/*
/sys/*
/dev/*
/tmp/*
/run/*
/mnt/*
/media/*
/var/log.hdd
/var/swap
```

**Rsync WILL cross filesystem boundries, so make sure you exclude external drives unless you want them included in the backup.**

Applications used in the script:
- fdisk (sfdisk)
- dd
- parted
- e2fsck
- truncate
- mkfs.ext4
- rsync

## Info

Theoretically the script should work on any device with the main storage on `/dev/mmcblk0` and maximum 2 partitions (boot and root).
The script can handle maximum 2 partitions, if there are more the script will not work.

### Order or operations - image creation
1. Reads the block sizes of the partitions
2. Uses `dd` to create the boot part of the system + a few megabytes to include the filesystem on root (this *can* be a partition)
3. Removes and recreates the root partition, size depends on options used when starting the script
4. Creates a new filesystem with the same UUID and LABEL as the system you are backing up from
5. Uses `rsync` to sync both partitions (if more than one)

This means it does not matter if boot is on a partition or f.ex uboot that Armbian uses.

Added space is added on top of `df` reported "used space", not the size of the partition. Added space is in MB, so if you want to add 1GB, add 1024.

The script can be instructed to set the img size by requesting recomended minimum size from `e2fsck` by using the `-a` option.
This is not the absolute smallest size you can achieve.
It is the "safest" way to create an img file. If you do not increase the size of the filesystem you are backing up too much, you can most likely keep it updated with the update function (`-U`) of the script.

To get the absolute smallest img file possible, do NOT set `-a` option and set "extra space" to 0

Example: `sudo shrink-backup /path/to/backup.img 0`

This will instruct the script to get the used space from `df` and adding 192MB "*wiggle room*".
If you are like me, doing a lot of testing, rewriting the sd-card multiple times. The extra time it takes each time will add up pretty fast.

Example:
```
-rw-r--r-- 1 root root 3.7G Jul 22 21:27 test.img # file created with -a
-rw-r--r-- 1 root root 3.3G Jul 22 22:37 test0.img # file created with 0
```

**Disclaimer:**
Because of how filesystems work, df is never a true representation of what will actually fit on a created img file.
Each file, no matter the size, will take up one block of the filesystem, so if you have a LOT of very small files (running docker f ex) the "0 added space method" might fail during rsync. Increase the 0 a little bit and retry.
This also means you have VERY little free space on the img file after creation.
If the filesystem you back up from increases in size, an update (-U) of the img file might fail.

### Order or operations - image update
1. Probes the img file for information about partitions
2. Mounts root partition with an offset for the loop
3. Checks if multiple partitions exists, if true, loops the boot with an offset and mounts it within the root mount
4. Uses `rsync` to sync both partitions (if more than one)

To update an existing img file simply use the `-U` option and the path to the img file.
Changing size in an update is not possible at the moment but is in the todo list for the future.

**Disclaimer**
If you only have one partition on your SBC (like armbian) updating the backup will NOT update the boot, only if you have 2 partitions is this possible. With 2 partition the boot will also be updated if files have changed.

The reason is the boot is not readable with the method used in the script, only dd would be able to achieve that.
If you want to update the boot in this scenario, you have to create a new img file.
As of this moment, there are no plans to include that functionality in the script.

This situation would happen if you update uboot on your armbian. I don't think it happens very often, but never the less good to keep in mind.

## To restore a backup, simply "burn" the img file to an sd-card using your favorite method.