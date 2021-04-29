---
#author: Fredrik Erlandsson
title: Migrate from one 512 GB SSD disk to a 1TB SSD disk on Synology DSM with no free disk slots
tags: synology, dsm, disk upgrade, md, mdadm
---

My "old" (Fri Oct 26 15:45:49 2018) 512 GB SSD disk have been alarming that it is running end of life so a new 1TB SSD disk will replace it.
_This is my own notes on what I did._

The initial stage is a 512 GB SSD in slot 1 and a 12 TB disk in slot 2. Both are configured as SHR.

## Stage 1, remove 12 TB disk

The Synology were shut down and the 12 TB disk was replaced with the 1 TB disk. This went as planned and both disks showed up and I added the new disk to the Storage pool which started to rebuild. After ~12 hours the rebuild was completed so I replaced the faulty 512 GB disk with the 12 TB disk (hotplug), and were hopefull that DSM would automatically detect everything on the 12 TB disk. Unfortunatly DSM didn't detect the Storage Pool on the big disk.  

## Stage 2, boot with just the 12TB disk

The Synology were shut down and the 1TB disk was removed. When the Synology was powered on again everything on the 12TB disk was showing. After some problem with accessing the Synology over SSH (I had ofcourse dissabled the `PasswordAuthentication` on sshd, and remounted the homes folder to space on the SSD. Which was fixed with `sed -i '/^PasswordAuthentication/{s/.*/#&/;:A;n;bA}' /etc/ssh/sshd_config` as a Task from DSM).

My thought here was that DSM didn't detect the LVMs on the 12 TB correctly when booted from the SSD system. DSM is using a md device for the system partitions. In stage 1 the system on the SSD couldn't detect the configuration correctly. However, the output from `vgdisplay -v` just returned `No volume groups found.`. So the next step was to look at what's mounted and try to make sense from that.

```console
root@Synology:/# mount
/dev/md0 on / type ext4 (rw,relatime,journal_checksum,barrier,data=ordered)
none on /dev type devtmpfs (rw,nosuid,noexec,relatime,size=4042584k,nr_inodes=1010646,mode=755)
none on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
none on /proc type proc (rw,nosuid,nodev,noexec,relatime)
none on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
/tmp on /tmp type tmpfs (rw,relatime)
/run on /run type tmpfs (rw,nosuid,nodev,relatime,mode=755)
/dev/shm on /dev/shm type tmpfs (rw,nosuid,nodev,relatime)
none on /sys/fs/cgroup type tmpfs (rw,relatime,size=4k,mode=755)
cgmfs on /run/cgmanager/fs type tmpfs (rw,relatime,size=100k,mode=755)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,relatime,cpuset,release_agent=/run/cgmanager/agents/cgm-release-agent.cpuset,clone_children)
cgroup on /sys/fs/cgroup/cpu type cgroup (rw,relatime,cpu,release_agent=/run/cgmanager/agents/cgm-release-agent.cpu)
cgroup on /sys/fs/cgroup/cpuacct type cgroup (rw,relatime,cpuacct,release_agent=/run/cgmanager/agents/cgm-release-agent.cpuacct)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,relatime,memory,release_agent=/run/cgmanager/agents/cgm-release-agent.memory)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,relatime,devices,release_agent=/run/cgmanager/agents/cgm-release-agent.devices)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,relatime,freezer,release_agent=/run/cgmanager/agents/cgm-release-agent.freezer)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,relatime,blkio,release_agent=/run/cgmanager/agents/cgm-release-agent.blkio)
none on /proc/bus/usb type devtmpfs (rw,nosuid,noexec,relatime,size=4042584k,nr_inodes=1010646,mode=755)
none on /sys/kernel/debug type debugfs (rw,relatime)
securityfs on /sys/kernel/security type securityfs (rw,relatime)
/dev/md2 on /volume2 type btrfs (rw,relatime,synoacl,space_cache=v2,auto_reclaim_space,metadata_ratio=50)
none on /config type configfs (rw,relatime)

root@Synology:/# cat /proc/partitions 
major minor  #blocks  name

   8       16 11718885376 sdb
   8       17    2490240 sdb1
   8       18    2097152 sdb2
   8       19 11714064384 sdb3
   9        0    2490176 md0
   9        1    2097088 md1
   9        2 11714063360 md2
 135      240     122880 synoboot
 135      241      16033 synoboot1
 135      242      96390 synoboot2

```

The interesting parts here are the `/dev/md` lines. So an investigation on mdadm resulted in this:

```console
root@Synology:/# cat /proc/mdstat 
Personalities : [linear] [raid0] [raid1] [raid10] [raid6] [raid5] [raid4] 
md2 : active raid1 sdb3[0]
      11714063360 blocks super 1.2 [1/1] [U]
      
md1 : active raid1 sdb2[1]
      2097088 blocks [2/1] [_U]
      
md0 : active raid1 sdb1[0]
      2490176 blocks [2/1] [U_]
      
unused devices: <none>

root@synology:/# mdadm --detail /dev/md*
/dev/md0:
        Version : 0.90
  Creation Time : Sat Jan  7 19:55:24 2017
     Raid Level : raid1
     Array Size : 2490176 (2.37 GiB 2.55 GB)
  Used Dev Size : 2490176 (2.37 GiB 2.55 GB)
   Raid Devices : 2
  Total Devices : 1
Preferred Minor : 0
    Persistence : Superblock is persistent

    Update Time : Wed Apr 28 13:26:30 2021
          State : clean, degraded 
 Active Devices : 1
Working Devices : 1
 Failed Devices : 0
  Spare Devices : 0

           UUID : faaf8e9e:d3c4d9b3:3017a5a8:c86610be
         Events : 0.1575021

    Number   Major   Minor   RaidDevice State
       0       8       17        0      active sync   /dev/sdb1
       -       0        0        1      removed
/dev/md1:
        Version : 0.90
  Creation Time : Wed Jul 10 12:21:48 2019
     Raid Level : raid1
     Array Size : 2097088 (2047.94 MiB 2147.42 MB)
  Used Dev Size : 2097088 (2047.94 MiB 2147.42 MB)
   Raid Devices : 2
  Total Devices : 1
Preferred Minor : 1
    Persistence : Superblock is persistent

    Update Time : Wed Apr 28 12:44:01 2021
          State : clean, degraded 
 Active Devices : 1
Working Devices : 1
 Failed Devices : 0
  Spare Devices : 0

           UUID : 56d6c602:91c22c01:c03470a7:6e080e63 (local to host SynHome)
         Events : 0.55

    Number   Major   Minor   RaidDevice State
       -       0        0        0      removed
       1       8       18        1      active sync   /dev/sdb2
/dev/md2:
        Version : 1.2
  Creation Time : Sun Jan 13 12:30:57 2019
     Raid Level : raid1
     Array Size : 11714063360 (11171.40 GiB 11995.20 GB)
  Used Dev Size : 11714063360 (11171.40 GiB 11995.20 GB)
   Raid Devices : 1
  Total Devices : 1
    Persistence : Superblock is persistent

    Update Time : Wed Apr 28 12:56:38 2021
          State : clean 
 Active Devices : 1
Working Devices : 1
 Failed Devices : 0
  Spare Devices : 0

           Name : SynHome:2  (local to host SynHome)
           UUID : 6ffcf44d:8e2b976b:97d1189f:883f3f08
         Events : 78

    Number   Major   Minor   RaidDevice State
       0       8       19        0      active sync   /dev/sdb3
```

What's interesting here is the fact that `/dev/sdb3` is configured as a raid1 device and is directly mounted on `/volume2`. My fear was that I needed to recreate the `md2` once the SSD was inserted again. So I shout down the Synology again and inserted the 1TB SSD.

## Stage 3, after reboot (12 TB disk in slot 2, 1 TB disk in slot 1)

Magically both disks show up. This time DSM used the system volume from the 12TB disk, so the system volume had to be restored on the SSD disk (from DSM).

The Storage Pool for the 1TB did however show up as degraded (as it was missing one disk). Back in with SSH and this is the result from mdadm:

```console
root@Synology:/# mdadm --detail /dev/md2
/dev/md2:
        Version : 1.2
  Creation Time : Fri Oct 26 15:45:49 2018
     Raid Level : raid1
     Array Size : 495274816 (472.33 GiB 507.16 GB)
  Used Dev Size : 495274816 (472.33 GiB 507.16 GB)
   Raid Devices : 2
  Total Devices : 1
    Persistence : Superblock is persistent

    Update Time : Wed Apr 28 15:15:00 2021
          State : clean, degraded 
 Active Devices : 1
Working Devices : 1
 Failed Devices : 0
  Spare Devices : 0

           Name : SynHome:2  (local to host SynHome)
           UUID : bf5d1657:09009110:465ee7eb:16143369
         Events : 105357

    Number   Major   Minor   RaidDevice State
       -       0        0        0      removed
       1       8        5        1      active sync   /dev/sda5
```

As shown one disk is missing (`removed`) and the State shows `clean, degraded`. `mdadm` can however be fixed for this with the `--grow -n 1` parameter.

```console
root@Synology:/# mdadm --grow -n 1 --force /dev/md2
raid_disks for /dev/md2 set to 1
root@Synology:/# mdadm --detail /dev/md2
/dev/md2:
        Version : 1.2
  Creation Time : Fri Oct 26 15:45:49 2018
     Raid Level : raid1
     Array Size : 495274816 (472.33 GiB 507.16 GB)
  Used Dev Size : 495274816 (472.33 GiB 507.16 GB)
   Raid Devices : 1
  Total Devices : 1
    Persistence : Superblock is persistent

    Update Time : Wed Apr 28 15:35:04 2021
          State : clean 
 Active Devices : 1
Working Devices : 1
 Failed Devices : 0
  Spare Devices : 0

           Name : SynHome:2  (local to host SynHome)
           UUID : bf5d1657:09009110:465ee7eb:16143369
         Events : 105362

    Number   Major   Minor   RaidDevice State
       1       8        5        0      active sync   /dev/sda5
```

Next part is to grow the Storage Pool so it uses the whole 1 TB of the disk. Here the issue was that the partition created by DSM only was ~500GB (to match the old disk). `parted` to the rescue:

```console
root@Synology:/# parted /dev/sda
GNU Parted 3.2
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) p                                                                
p
Model: Seagate IronWolf ZA1000N (scsi)
Disk /dev/sda: 1000GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type      File system  Flags
 1      1049kB  2551MB  2550MB  primary                raid
 2      2551MB  4699MB  2147MB  primary                raid
 3      4832MB  1000GB  995GB   extended               lba
 5      4840MB  512GB   507GB   logical                raid

(parted) resizepart                                                       
resizepart
Partition number? 5                                                       
5
End?  [512GB]? 1000GB                                                     
1000GB
(parted) print                                                            
print
Model: Seagate IronWolf ZA1000N (scsi)
Disk /dev/sda: 1000GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type      File system  Flags
 1      1049kB  2551MB  2550MB  primary                raid
 2      2551MB  4699MB  2147MB  primary                raid
 3      4832MB  1000GB  995GB   extended               lba
 5      4840MB  1000GB  995GB   logical                raid

(parted) quit                                                             
quit
Information: You may need to update /etc/fstab.
```

Magics made in DSM (grow Storage Pool).

Result:

```console
root@Synology:/# mdadm --detail /dev/md2
/dev/md2:
        Version : 1.2
  Creation Time : Fri Oct 26 15:45:49 2018
     Raid Level : raid1
     Array Size : 971834816 (926.81 GiB 995.16 GB)
  Used Dev Size : 971834816 (926.81 GiB 995.16 GB)
   Raid Devices : 1
  Total Devices : 1
    Persistence : Superblock is persistent

    Update Time : Wed Apr 28 15:59:59 2021
          State : clean 
 Active Devices : 1
Working Devices : 1
 Failed Devices : 0
  Spare Devices : 0

           Name : SynHome:2  (local to host SynHome)
           UUID : bf5d1657:09009110:465ee7eb:16143369
         Events : 105365

    Number   Major   Minor   RaidDevice State
       1       8        5        0      active sync   /dev/sda5
```

## Final stage

`md` detail:

```console
root@Synology:/# mdadm --detail /dev/md*
/dev/md0:
        Version : 0.90
  Creation Time : Sat Jan  7 19:55:24 2017
     Raid Level : raid1
     Array Size : 2490176 (2.37 GiB 2.55 GB)
  Used Dev Size : 2490176 (2.37 GiB 2.55 GB)
   Raid Devices : 2
  Total Devices : 2
Preferred Minor : 0
    Persistence : Superblock is persistent

    Update Time : Thu Apr 29 09:27:09 2021
          State : clean 
 Active Devices : 2
Working Devices : 2
 Failed Devices : 0
  Spare Devices : 0

           UUID : faaf8e9e:d3c4d9b3:3017a5a8:c86610be
         Events : 0.1581285

    Number   Major   Minor   RaidDevice State
       0       8       17        0      active sync   /dev/sdb1
       1       8        1        1      active sync   /dev/sda1
/dev/md1:
        Version : 0.90
  Creation Time : Wed Apr 28 15:09:35 2021
     Raid Level : raid1
     Array Size : 2097088 (2047.94 MiB 2147.42 MB)
  Used Dev Size : 2097088 (2047.94 MiB 2147.42 MB)
   Raid Devices : 2
  Total Devices : 2
Preferred Minor : 1
    Persistence : Superblock is persistent

    Update Time : Thu Apr 29 09:10:54 2021
          State : clean 
 Active Devices : 2
Working Devices : 2
 Failed Devices : 0
  Spare Devices : 0

           UUID : 74eedd98:b2299432:c03470a7:6e080e63 (local to host SynHome)
         Events : 0.2

    Number   Major   Minor   RaidDevice State
       0       8        2        0      active sync   /dev/sda2
       1       8       18        1      active sync   /dev/sdb2
/dev/md2:
        Version : 1.2
  Creation Time : Fri Oct 26 15:45:49 2018
     Raid Level : raid1
     Array Size : 971834816 (926.81 GiB 995.16 GB)
  Used Dev Size : 971834816 (926.81 GiB 995.16 GB)
   Raid Devices : 1
  Total Devices : 1
    Persistence : Superblock is persistent

    Update Time : Thu Apr 29 09:27:09 2021
          State : clean 
 Active Devices : 1
Working Devices : 1
 Failed Devices : 0
  Spare Devices : 0

           Name : SynHome:2  (local to host SynHome)
           UUID : bf5d1657:09009110:465ee7eb:16143369
         Events : 105365

    Number   Major   Minor   RaidDevice State
       1       8        5        0      active sync   /dev/sda5
/dev/md3:
        Version : 1.2
  Creation Time : Sun Jan 13 12:30:57 2019
     Raid Level : raid1
     Array Size : 11714063360 (11171.40 GiB 11995.20 GB)
  Used Dev Size : 11714063360 (11171.40 GiB 11995.20 GB)
   Raid Devices : 1
  Total Devices : 1
    Persistence : Superblock is persistent

    Update Time : Thu Apr 29 09:08:37 2021
          State : clean 
 Active Devices : 1
Working Devices : 1
 Failed Devices : 0
  Spare Devices : 0

           Name : SynHome:2  (local to host SynHome)
           UUID : 6ffcf44d:8e2b976b:97d1189f:883f3f08
         Events : 82

    Number   Major   Minor   RaidDevice State
       0       8       19        0      active sync   /dev/sdb3
```

Volume groups:

```console
root@Synology:/# vgdisplay 
  --- Volume group ---
  VG Name               vg1
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  6
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               1
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               926.81 GiB
  PE Size               4.00 MiB
  Total PE              237264
  Alloc PE / Size       237059 / 926.01 GiB
  Free  PE / Size       205 / 820.00 MiB
  VG UUID               ijkBYn-GVFa-jBoI-vbnW-i3Du-OzQs-gbOhrH
```

LVM backup.

```config
# Generated by LVM2 version 2.02.132(2)-git (2015-09-22): Thu Apr 29 09:35:38 2021

contents = "Text Format Volume Group"
version = 1

description = "Created *after* executing 'vgcfgbackup'"

creation_host = "Synology" # Linux SynHome 3.10.105 #25426 SMP Tue May 12 04:44:40 CST 2020 x86_64
creation_time = 1619681738	# Thu Apr 29 09:35:38 2021

vg1 {
	id = "ijkBYn-GVFa-jBoI-vbnW-i3Du-OzQs-gbOhrH"
	seqno = 6
	format = "lvm2"			# informational
	status = ["RESIZEABLE", "READ", "WRITE"]
	flags = []
	extent_size = 8192		# 4 Megabytes
	max_lv = 0
	max_pv = 0
	metadata_copies = 0

	physical_volumes {

		pv0 {
			id = "qOE9fN-udnH-G5de-YlgN-XAAq-WtO3-MDdePc"
			device = "/dev/md2"	# Hint only

			status = ["ALLOCATABLE"]
			flags = []
			dev_size = 1943668480	# 926.813 Gigabytes
			pe_start = 1152
			pe_count = 237264	# 926.812 Gigabytes
		}
	}

	logical_volumes {

		syno_vg_reserved_area {
			id = "pWpvXH-u22k-5cpX-geom-byeF-REek-MRWlPn"
			status = ["READ", "WRITE", "VISIBLE"]
			flags = []
			segment_count = 1

			segment1 {
				start_extent = 0
				extent_count = 3	# 12 Megabytes

				type = "striped"
				stripe_count = 1	# linear

				stripes = [
					"pv0", 0
				]
			}
		}

		volume_1 {
			id = "UD1bl0-ZzXF-Djf9-tJKi-02gT-fzP4-N19VKU"
			status = ["READ", "WRITE", "VISIBLE"]
			flags = []
			segment_count = 1

			segment1 {
				start_extent = 0
				extent_count = 237056	# 926 Gigabytes

				type = "striped"
				stripe_count = 1	# linear

				stripes = [
					"pv0", 3
				]
			}
		}
	}
}
```
