# redundantfilestorage
Brainstorming for the requirements for storing files in multiple hard drives with redundancy and fault tolerance


# Inspiration

brtfs raid 5/6 modes, which after a few tests by me using Ubuntu 17.10 beta on 2017-09-19 seems to be broken even after the 2017-08-01 patch (see https://btrfs.wiki.kernel.org/index.php/RAID56)

Specifically:
- create a VM with 4 aditional disks
- create a btrfs volume with data in raid 5 and metadata in raid6 using all 4 disks
- write a few GB of random bytes in the btrfs volume
- shutdown the vm
- disconnect one of the disks from the vm
- start the vm
- mount volume in degraded mode
- run btrfs device remove <the disk you disconnected>
- run btrfs balance start <volume mount point>
- volume should be fine at this point
- shutdown the vm
- reconnect the disk you disconnected before (it still thinks it is part of the volume)
- start the vm
- mount the volume on a folder
- try to add the disconnected disk back to the volume. Will ask you to use the -f flag to overwrite the brtfs in the disk.
- after trying to add, will still refuse saying device IS mounted.
- try to run btrfs device remove <the disk you disconnected>. Will fail saying the device IS NOT mounted.
- this part I do not remember exactly, but I think I zeroed the first 20 MB of the disk I had disconnected and then successfully added it to the mounted volume.
- begin writing some more random data to the supposedly ok mounted volume
- simultaneously, in another terminal, run btrfs balance start on the volume
- machine will lock up, unresponsive to CTRL+ALT+DEL, alt+f2 and the like.
- I did not test what happens to the data after a hard reset.
  
I know, this test is a bit too extreme, but that is similar to the condition I expect to find if I mount remote disks over iscsi and one of them disconnects temporarily. I want the rest of my array to heal and resize gracefully, and if the extra disk is reconnected, I want to be able to recover the capacity by adding it back, correctly handling the inconsistent btrfs state the reconnected media is.


# Requirements for an alternate solution

* Work on Linux. Best if also works on BSD/OSX/Windows.
* Able to store files in several physical or logical devices
* may be on top of another filesystem or even SAMBA (N separate local disks, each formated with ext4 for example, or N samba shares mounted to N local folders or another mix of FUSE mountable stuff)

For example, chunks may be stored in regular files and metadata in a sqlite database, although not quite like that because I assume there has to me some mechanism to guarantee transactions and a single sqlite file would be both a single point of failure and probably unable to serve more than one simultaneous connection to the metadata file.

* gracefully handle device failures up to M=N-1 devices, where N is the number of devices used and M is a redundancy factor 
* gracefully handle device disconnect during writes then reconnected later
* automatically self heal and reduce volume capacity when a device fails, provided that there is enough capacity left to hold the data at the required redundancy.
* gracefully handle cascaded device failures where one device fails after the other before the array has a chance to fully heal and recover M redundancy. For example: N=10 and M=3, a device fails and volume starts to regenerate to N=9 and M=3; a second disk fails before the volume is healed and then a third disk fails. If enough free space is available the volume should successfully regenerate to N=7 and M=3.
* gracefully handle device reconnects during self healing. If a volume with N=10 and M=3 suffers a device disconnect, the volume may have a grace time to try to reconnect (this will deny service by holding reads and writes while trying to reconnect) then it will begin to self heal. If the device reconnects right after, gracefully handle that (ideally it should try to use as much as possible of the data already on the volume).
* support custom tests or metrics to voluntarily flag a device as bad (too many failed consistency checks, too many disconnects, too much latency, etc.)
* support data and metadata checksums
* support assimetric devices on the same volume (low/high latency, low/high bandwidth, different capacities).
* gracefully decrease redundancy level and self rebuild when a drive fails. For example, in a volume where N=3 (3 devices) and N=2 (2 drives can fail and data can be preserved), if a device fail the volume will be rebuilt with N=1. Or if M=4, N=2 and a drive fails but the data in the array cannot be held with one drive capacity, the volume degrades to M=3, N=1.
* provide a means to push alerts of degradation
* dead simple to deploy. Much simpler than hdfs or ceph. Ideally a single executable with a configuration file. Multiple deployments with the same configuration file would generate a multimaster redundant service.
* mountable via FUSE (web api not required, can be shared using regular samba or NFS from the fuse mount)
* preferably support for multiple simultaneous clients
* dead simple to understand and audit. There are too many distributed filesystems already. It has to be something people are not afraid to use because it's too new and untested - alternatively, it can be so simple to understand and audit that it will become well tested very soon.
* ideally be able to store all the configuration and metadata together on the devices. The configuration file of the client should be minimal to perform an initial connection to any one of the devices and retrieve the full configuration.
* support chunk or file deduplication
* support scrubbing
* support manual substitution of a device for another (hd SMART parameters with fail threat)
* maybe support automatic substitution of a device for another (hot standby) (is this even necessary after the cascaded failure requirements?)
* support snapshots, only latest snapshot is required to be writable.
* support mounting snapshots read-only.
* support snapshot rollback.
* support intermediary snapshot deletion.
