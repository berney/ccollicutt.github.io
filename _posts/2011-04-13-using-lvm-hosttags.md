---
layout: post
title: Using LVM hosttags
categories:
- serverascode
---

# {{ page.title }}

This is a somewhat minor post, but I thought it would be worthwhile to take a peek at using LVM hosttags to manage dom0 access to logical volumes on top of SAN LUNs because there doesn't seem to be a lot of documentation on using hosttags online. Perhaps that's because no one is doing it this way. 

While it's not my favorite way of managing SAN disks across servers (I like [clvmd](http://docs.redhat.com/docs/en-US/Red_Hat_Enterprise_Linux/6/html/Logical_Volume_Manager_Administration/LVM_Cluster_Overview.html) but it brings considerable complexity), hosttags are certainly one way to do it. Hosttags are a relatively simple method, and better than LVM filters IMHO. The point of using LVM hosttags is to ensure that only one server is ever writing to a SAN LUN at a time (unless you have something like GFS or similar in use, which is managing that process of multiple writers, and in that case what are you doing here? :) ).

In this example we have three Redhat Enterprise 5.x dom0 servers connected to a large--and expensive--storage area network. We'll call them `vmhost1`, `vmhost2`, and `vmhost3`. The LUNs are provided to the vmhosts via the SANs configuration software. The vmhosts see them as regular disk, but they _aren't_ regular disk because each of the servers can see them, where see means read and write. (Note that I may be using terms incorrectly, but by SAN LUN I mean the slice of SAN disk provided to the server over fibre channel.)

<pre>
<code>[root@vmhost1 ~]# multipath -l | grep 00000002
mpath3 (877880e80144455000001445500000002) dm-12 HITACHI,OPEN-V*4
[root@vmhost2 ~]# multipath -l | grep 00000002
mpath3 (877880e80144455000001445500000002) dm-11 HITACHI,OPEN-V*4
[root@vmhost3 ~]# multipath -l | grep 00000002
mpath3 (877880e80144455000001445500000002) dm-11 HITACHI,OPEN-V*4
</code>
</pre>

As is shown in the above output, each of the hosts can see the same SAN LUN: `877880e80144455000001445500000002` which in each case is also called `mpath3`. But I don't really care about what names the disk is given because I run LVM on top of those disks.

<pre>
<code>[root@vmhost2 ~]# pvs | grep mpath3
  /dev/mapper/mpath3 some_volume_group LVM2 a-    96.62G  16.62G
</code>
</pre>

So, we have a SAN LUN, a physical volume (PV) made from that LUN, and a volume group (VG) called `some_volume_group` created from that PV.

So it sort of looks like this in terms of hierarchy:

- Fibre Channel SAN LUN
- multipathd
- PV
- VG
- Logical volume (LV) which has hosttags assigned


## Using hosttags

First we make sure we have hosttags configured on each of the vmhosts.

<pre>
<code>[root@vhmost2 lvm]# pwd
/etc/lvm
[root@vmohost2 lvm]# grep hosttags lvm.conf
tags { hosttags = 1 }
</code>
</pre>

Obviously because I run using LVM hosttags in production, I already have hosttags configured and being used to control vmhost access to LVs.

<pre>
<code>[root@vmohost2 lvm]# uname -n
xmhost2.example.com
[root@vmhost2 lvm]# lvdisplay @`uname -n` | grep "LV Name"
SNIP!
  lv Name                /dev/some_logical_volume/test
</code>
</pre>

If I want to create a LV on a vmhost, and that VG is configured to use hosttags, then it has to be done properly. This is an example that will fail because the host does not have permission, ie. the LV is not available because of the lack of a hosttag attribute on the LV that names the @uname -n@ host specifically.

<pre>
<code>[root@vmhost2 ~]# lvcreate -n test2 -L10.0G /dev/some_volume_group
# Will fail out with error message
</code>
</pre>

But this _next_ command will work, because we are saying create a LV and assign a hosttag to it which is the same as the hostname that is creating it. The vmhost can't create a LV if it's not made available via a hosttag.

<pre>
<code>[root@vmhost2 ~]# uname -a
vmhost2.example.com
[root@vmhost2 ~]# lvcreate --addtag @vmhost2.example.com -n test2 \
-L10.0G /dev/some_volume_group
</code>
</pre>

Now, on `vmhost1` we can see that the LV appears in the list, but it is not available:

<pre>
<code>[root@vmhost1 ~]# lvs some_volume_group | grep test2
  test2            some_volume_group -wi--- 10.00G
</code>
</pre>

But it's available on vmhost2, where it was created, and where it has a hosttag attribute of `vmhost2.example.com`.

<pre>
<code>[root@vmhost2 ~]# lvs some_volume_group | grep test2 
  test2            some_volume_group -wi-a- 10.00G
</code>
</pre>

And, if you use @lvdisplay @`uname -n`@ you can see what LVs have the servers @uname -n@ tag:

<pre>
<code>[root@vmhost2 ~]# lvdisplay @vmhost2.example.com | grep "LV Name"
  LV Name                /dev/some_volume_group/test2
</code>
</pre>

whereas `vmhost1` does not see that LV:

<pre>
<code>[root@vmhost1 ~]# lvdisplay @vmhost1.example.com | grep "LV Name"
# Nothing returns, as expected
</code>
</pre>

So, while this sounds complicated, it really isn't. Essentially for a LV to be available to a server that can see the same LUNs as other servers, in terms of LVM, it must have its hosttag added to the LVs metadata. Otherwise, it's not available and can't be used on that host.

While all three hosts can see the LV, it's only available to those vmhosts that have had their hostname added to the specific LVs tags. It adds complexity to the vmhost setup and use, but it's better to do this than to end up having two virtual machines writing to the same LV. Other options include using filtering in LVM, or even going all the way and using [clvmd](http://docs.redhat.com/docs/en-US/Red_Hat_Enterprise_Linux/6/html/Logical_Volume_Manager_Administration/LVM_Cluster_Overview.html), which I have done, _but that's another story..._ :)
