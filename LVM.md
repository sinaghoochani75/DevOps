# LVM Disk Expansion on Ubuntu 24.04

## 1. Introduction

This guide explains how to extend an existing Linux storage configuration using Logical Volume Manager (LVM).

In this scenario, we add a new 250 GB virtual disk to an Ubuntu 24.04 virtual machine running on VMware ESXi. Then, we integrate the new disk into the existing LVM Volume Group and distribute the available storage between the root (`/`) and `/var` Logical Volumes.

### Environment Specifications

| Component | Value |
|---|---|
| Operating System | Ubuntu 24.04 |
| Hypervisor | VMware ESXi |
| Original Disk | `/dev/sda` |
| New Disk | `/dev/sdb` |
| New Disk Capacity | 250 GB |
| New Partition | `/dev/sdb1` |
| Volume Group | `ubuntu-vg` |
| Root Logical Volume | `ubuntu-lv` |
| Var Logical Volume | `lv-0` |
| Filesystem | ext4 |

> **Warning:** Disk names and Logical Volume paths may differ between systems. Always verify your storage configuration before executing commands. Back up important data before modifying disk partitions or LVM structures.

---

## 2. Adding a New Virtual Disk in VMware ESXi

First, log in to the VMware ESXi management interface.

Follow these steps:

1. Select the Ubuntu virtual machine.
2. Open **Edit Settings**.
3. Select **Add Hard Disk**.
4. Set the disk capacity to `250 GB`.
5. Select `datastore2` as the storage location.
6. Save the configuration.

After adding the virtual disk, log in to the Ubuntu operating system.

---

## 3. Detecting the New Disk in Linux

First, identify the available disks:

```bash
lsblk
```

To display detailed disk information:

```bash
sudo fdisk -l
```

To inspect the newly added disk:

```bash
sudo fdisk -l /dev/sdb
```

In this scenario, the new disk should appear as `/dev/sdb` with a capacity of approximately 250 GB.

Example:

```text
NAME                      SIZE TYPE MOUNTPOINTS
sda                        50G disk
├─sda1                      1G part /boot
└─sda2                     49G part
  └─ubuntu-vg
    ├─ubuntu-lv             30G lvm  /
    └─lv-0                  10G lvm  /var

sdb                       250G disk
```

If the disk does not appear, verify its configuration in VMware ESXi.

You can also trigger a SCSI rescan:

```bash
for host in /sys/class/scsi_host/host*; do
    echo "- - -" | sudo tee "$host/scan"
done

lsblk
```

---

## 4. Partitioning the New Disk

Before using the new disk with LVM, we need to create a partition.

Start the `fdisk` utility:

```bash
sudo fdisk /dev/sdb
```

Enter the following commands:

```text
g
n
1
Enter
Enter
w
```

Command explanations:

| Command | Description |
|---|---|
| `g` | Create a GPT partition table |
| `n` | Create a new partition |
| `1` | Set the partition number |
| `Enter` | Accept the default first sector |
| `Enter` | Use the remaining available disk space |
| `w` | Write changes and exit |

After completing these steps, the new partition should be `/dev/sdb1`.

Notify the Linux kernel about the partition table changes:

```bash
sudo partprobe /dev/sdb
```

Verify the result:

```bash
lsblk
```

Expected output:

```text
sdb       250G disk
└─sdb1    250G part
```

---

## 5. Creating a Physical Volume (PV)

The first LVM step is converting the newly created partition into a Physical Volume.

Execute:

```bash
sudo pvcreate /dev/sdb1
```

Verify the Physical Volume:

```bash
sudo pvs
```

For detailed information:

```bash
sudo pvdisplay /dev/sdb1
```

The partition `/dev/sdb1` is now available as an LVM Physical Volume.

---

## 6. Extending the Volume Group (VG)

In this environment, the existing Volume Group is named `ubuntu-vg`.

Add the new Physical Volume to the existing Volume Group:

```bash
sudo vgextend ubuntu-vg /dev/sdb1
```

Verify the Volume Group:

```bash
sudo vgs
```

For detailed information:

```bash
sudo vgdisplay ubuntu-vg
```

The new disk capacity is now available within the existing Volume Group.

At this stage, the additional storage has not yet been assigned to the root or `/var` Logical Volumes.

---

## 7. Identifying Existing Logical Volumes

Before extending the Logical Volumes, inspect the current LVM configuration:

```bash
sudo lvs
```

For detailed information:

```bash
sudo lvdisplay
```

In this scenario, we have two Logical Volumes:

| Mount Point | Logical Volume |
|---|---|
| `/` | `/dev/ubuntu-vg/ubuntu-lv` |
| `/var` | `/dev/ubuntu-vg/lv-0` |

Check the filesystem configuration:

```bash
df -hT
```

This command displays filesystem types, sizes, used space, available space, and mount points.

---

## 8. Extending the Root Logical Volume

In this step, we increase the root Logical Volume to a final size of 197 GB.

Execute:

```bash
sudo lvextend -L 197G /dev/ubuntu-vg/ubuntu-lv
```

The `-L` option specifies the final Logical Volume size.

### Difference Between `-L` and `-l`

Setting the final Logical Volume size:

```bash
sudo lvextend -L 197G /dev/ubuntu-vg/ubuntu-lv
```

Adding 197 GB to the current Logical Volume size:

```bash
sudo lvextend -L +197G /dev/ubuntu-vg/ubuntu-lv
```

In this scenario, we use the first command.

Verify the new Logical Volume size:

```bash
sudo lvs
```

> Make sure the Volume Group has enough free space before extending the Logical Volume.

---

## 9. Resizing the Root Filesystem

Increasing the Logical Volume size does not automatically guarantee that the filesystem can use the additional capacity.

Since the root filesystem uses ext4, we use `resize2fs` to expand it.

Execute:

```bash
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

This command expands the ext4 filesystem to use the available Logical Volume capacity.

Verify the result:

```bash
df -h /
```

---

## 10. Assigning the Remaining Space to /var

After extending the root Logical Volume, we assign the remaining free space in the Volume Group to the `/var` Logical Volume.

First, check the available free space:

```bash
sudo vgs
```

Then execute:

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/lv-0
```

Parameter explanations:

| Parameter | Description |
|---|---|
| `-l` | Specifies the size in Logical Extents |
| `+` | Adds capacity to the existing Logical Volume |
| `100%FREE` | Uses all remaining free space in the Volume Group |

This command allocates all remaining free extents to the `/var` Logical Volume.

> **Important:** Using `100%FREE` consumes all remaining free space in the Volume Group. If you need storage capacity for future expansion, allocate only a specific amount instead.

---

## 11. Resizing the /var Filesystem

The next step is expanding the filesystem associated with `/var`.

Since the filesystem is ext4, execute:

```bash
sudo resize2fs /dev/ubuntu-vg/lv-0
```

The filesystem can now use the additional storage capacity.

Verify the result:

```bash
df -h /var
```

---

## 12. Verifying the Final LVM Configuration

After completing all operations, verify the entire storage configuration.

### Check Disks and Partitions

```bash
lsblk
```

### Check Physical Volumes

```bash
sudo pvs
```

### Check Volume Groups

```bash
sudo vgs
```

### Check Logical Volumes

```bash
sudo lvs
```

### Check Filesystems

```bash
df -hT
```

### Display Detailed Logical Volume Information

```bash
sudo lvdisplay
```

The following conditions should be satisfied:

- The new disk `/dev/sdb` is detected.
- The partition `/dev/sdb1` is successfully created.
- The partition is initialized as a Physical Volume.
- The new Physical Volume is added to `ubuntu-vg`.
- The root Logical Volume is extended to its intended size.
- The remaining free space is assigned to the `/var` Logical Volume.
- Both ext4 filesystems are successfully resized.

---

## 13. Final LVM Architecture

The final storage architecture looks like this:

```text
/dev/sdb
   |
   +-- /dev/sdb1
          |
          +-- Physical Volume (PV)
                    |
                    +-- ubuntu-vg (Volume Group)
                            |
                            +-- ubuntu-lv (Logical Volume)
                            |       |
                            |       +-- ext4
                            |               |
                            |               +-- /
                            |
                            +-- lv-0 (Logical Volume)
                                    |
                                    +-- ext4
                                            |
                                            +-- /var
```

---

## 14. Additional Notes

### LVM vs. Filesystem

LVM manages storage at the block-device level, while filesystems such as ext4 organize files and directories within the available storage.

Therefore, after increasing a Logical Volume, the filesystem must also be expanded to use the additional capacity.

### ext4 vs. XFS

For ext4 filesystems, use:

```bash
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

For XFS filesystems, use `xfs_growfs` with the mount point.

For example:

```bash
sudo xfs_growfs /var
```

You can identify the filesystem type using:

```bash
df -hT
```

### Extending the Logical Volume and Filesystem Together

LVM provides the `-r` option to extend the Logical Volume and resize the filesystem in a single operation.

Example:

```bash
sudo lvextend -r -L 197G /dev/ubuntu-vg/ubuntu-lv
```

In this guide, we performed these operations separately to make each step easier to understand.

---

## 15. Command Summary

All the main commands used in this scenario are listed below in execution order.

```bash
# 1. Identify available disks
lsblk
sudo fdisk -l

# 2. Partition the new disk
sudo fdisk /dev/sdb

# 3. Notify the kernel about partition changes
sudo partprobe /dev/sdb

# 4. Create a Physical Volume
sudo pvcreate /dev/sdb1

# 5. Extend the existing Volume Group
sudo vgextend ubuntu-vg /dev/sdb1

# 6. Check Volume Group status
sudo vgs

# 7. Check existing Logical Volumes
sudo lvs

# 8. Extend the root Logical Volume
sudo lvextend -L 197G /dev/ubuntu-vg/ubuntu-lv

# 9. Resize the root filesystem
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv

# 10. Allocate remaining free space to /var
sudo lvextend -l +100%FREE /dev/ubuntu-vg/lv-0

# 11. Resize the /var filesystem
sudo resize2fs /dev/ubuntu-vg/lv-0

# 12. Verify the final configuration
lsblk
sudo pvs
sudo vgs
sudo lvs
df -hT
```

---

## Conclusion

In this scenario, we added a new 250 GB virtual disk to an Ubuntu 24.04 virtual machine running on VMware ESXi.

We partitioned the disk, initialized it as an LVM Physical Volume, and added it to the existing Volume Group.

Finally, we distributed the available storage between the root and `/var` Logical Volumes and expanded both ext4 filesystems using `resize2fs`.

This approach allows Linux administrators to increase existing filesystem capacity without creating a separate filesystem on the newly added disk.

LVM provides flexible and centralized storage management, making it easier to expand storage resources as system requirements grow.