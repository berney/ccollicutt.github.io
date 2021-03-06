---
layout: post
title: Truncate command and sparse disks
categories:
---

# {{ page.title }}

Just a quick post on creating sparse disks on Linux. 

Currently I am working on deploying [OpenStack Swift](http://docs.openstack.org/developer/swift/). I am using Vagrant and Virtualbox virtual machines to create an [Ansible](http://ansibleworks.com) playbook to deploy and manage Swift. To do that I have been using sparse disks to mimic real disks without actually having all the disk space required.

<pre>
<code>$ cd /var/tmp

# Create a sparse file using the truncate command
$ truncate --size 500G sparse_disk1.img
$ ls -la sparse_disk1.img 
-rw-rw-r-- 1 vagrant vagrant 536870912000 Nov 11 06:20 sparse_disk1.img

# Check if there is a loop0 already...
$ sudo losetup /dev/loop0
loop: can't get info on device /dev/loop0: No such device or address

# Ok good lets setup the loop device with the sparse disk image
$ sudo losetup /dev/loop0 /var/tmp/sparse_disk1.img 

# Make a file system on that loop device
$ sudo mkfs.xfs -i size=1024 /dev/loop0
meta-data=/dev/loop0             isize=1024   agcount=4, agsize=32768000 blks
         =                       sectsz=512   attr=2, projid32bit=0
data     =                       bsize=4096   blocks=131072000, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0
log      =internal log           bsize=4096   blocks=64000, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

# And mount it
$ sudo mkdir /mnt/sparse_disk1
$ sudo mount -o noatime,nodiratime,nobarrier /dev/loop0 /mnt/sparse_disk1
$ df -h /dev/loop0
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0      500G   33M  500G   1% /mnt/sparse_disk1
</code>
</pre>

To unmount/remove...just go in reverse. :)

<pre>
<code>$ sudo umount /mnt/sparse_disk1
$ sudo losetup -d /dev/loop0
$ rm -f /var/tmp/sparse_disk1.img 
</code>
</pre>

If you haven't used Vagrant and configuration management systems such as Chef, Puppet, Ansible, Salt, etc, I highly suggest it. Putting up and tearing down servers and clusters of servers gets addicting. 
