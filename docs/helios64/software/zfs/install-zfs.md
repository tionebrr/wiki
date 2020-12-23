!!! Important
    This install procedure only works with *Armbian Focal* for now. Instructions for *Armbian Buster* to be added soon.

So you already installed the system on eMMC or SD? You might want to use ZFS on the hard disk(s)! We assume rootfs is already on eMMC (or microSD Card) and you want to store your data on HDDs in ZFS pool.

!!! Note
    This wiki does not cover root-on-zfs. (Although it should be also possible.)



##  **Step 1** - Install ZFS

```bash
sudo armbian-config
```

Go to *Software* and install the headers.

![Kernel Headers](/helios64/software/zfs/img/install-headers.png)

Once the kernel headers are installed, install ZFS with the following command:

```bash
sudo apt install zfs-dkms zfsutils-linux
```

Optional:

```bash
sudo apt install zfs-auto-snapshot
```

Reboot.

##  **Step 2** - Prepare partitions (optional)

Use `fdisk` of `gdisk` to create the necessary partitions on your hard drive. Some details on why you might need more partitions can be found in this armbian forum thread: https://forum.armbian.com/topic/16559-tutorial-first-steps-with-helios64-zfs-install-config/?do=findComment&comment=116028.

##  **Step 3** - Create ZFS pool

ZFS pool should not be created using `/dev/sdXY` names because those symlinks might not point to the exact same partition after each reboot. All disks and partitions in a ZFS pool must be unique and identified as such. This way your system will still work when you remove a disk or change the order of your disks. This could be achieved by several methods:

### Method 1: create by partuuid
This method is the recommended method until further testing and validation.
When ready, look for assigned uuids:

```bash
ls -l /dev/disk/by-partuuid/
```

```bash
sudo zpool create -o ashift=12 -m /mypool mypool mirror /dev/disk/by-partuuid/<diskA-uuid> /dev/disk/by-partuuid/<diskB-uuid>
sudo zfs set atime=off mypool
sudo zfs set compression=on mypool
```

If your disks are SSDs, enable trim support:
```bash
sudo zpool set autotrim=on mypool
```

Of course you may use more disks and create raidz instead of mirror. Your choice.
Here is a link to a ZFS quick start guide:
https://wiki.debian.org/ZFS#Creating_the_Pool

### Method 2: Create with /dev/sdX and reimport
We're going to create a pool the bad way first:
```bash
sudo zpool create -o ashift=12 -m /mypool mypool mirror /dev/<sdXp> /dev/<sdYp>
sudo zfs set atime=off mypool
sudo zfs set compression=on mypool
```
If you partitionned your disks, don't forget to indicate the partition numbers you reserved for ZFS when calling `zpool create`.

Now, if you type `zpool status`, you will see that your disks are referenced by /dev/sdX paths; not good... We need to change those references to UUID; and that could be made quite easily by exporting and reimporting the pool.
```bash
sudo zpool export mypool # This will unmount your ZFS pool and deactivate it in your system, but the pool informations are still on the disks
sudo zpool import -d /dev/disk/by-partuuid mypool  # This will search for the pool named 'mypool' in the /dev/disk/by-partuuid directory
```

If you now look at `zpool status`, you disks should now be referenced by partition UUID.

Note: Zpool might not be able to unmount the zfs pool during export if you already have services running and using files or directory from the pool. That will be the case with docker if you are already storing volumes in your pool.
To work around this, docker must be shut down:
```bash
docker ps # Get a list of your running containers
docker stop <containerA> <containerB> etc...
sudo service docker stop # An now you can export and re-import
```

Optionally, you can import back your pool using `/disk/by-id` so that the NAME of your disks are based on the manufacturer Reference and Serial Number. There is one catch however. The /dev/disk/by-id contains wwn-* references that are not really helpful when it comes to identifying disks. The simplest way I've found to workaround this problem is to just delete the wwn-* references and only then import your pool by-id.

```bash
sudo export mypool
sudo rm /dev/disk/by-id/wwn*
sudo zpool import -d /dev/disk/by-id mypool
```
The wwn symlinks will be back after a reboot.

`zpool status` output should now look like this:
```bash
$> zpool status
  pool: tank
 state: ONLINE
  scan: scrub repaired 0B in 0 days 00:14:20 with 0 errors on Tue Dec 22 22:30:46 2020
config:

        NAME                              STATE     READ WRITE CKSUM
        tank                              ONLINE       0     0     0
          raidz2-0                        ONLINE       0     0     0
            ata-HUA721010KLA330_PBKJ9PDF  ONLINE       0     0     0
            ata-HUA721010KLA330_PBKJK34F  ONLINE       0     0     0
            ata-ST31000340NS_9QJ7YY0Z     ONLINE       0     0     0
            ata-ST31000340NS_9QJ841MF     ONLINE       0     0     0

errors: No known data errors
```

##  **Step 4** - Reboot your system

If everything went well, your pool should be imported automatically after reboots:
```bash
zpool status
```

You now have a working system with root on eMMC and a ZFS pool on HDD.

## **Further readings**:
https://wiki.debian.org/ZFS

------------

*Page contributed by [michabbs](https://github.com/michabbs)*

*Reference [Armbian Forum Dicussion](https://forum.armbian.com/topic/16559-tutorial-first-steps-with-helios64-zfs-install-config/)*
