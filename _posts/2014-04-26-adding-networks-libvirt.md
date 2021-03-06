---
layout: post
title: Adding networks to libvirt
categories: 
header_image: /img/met/DP821057.jpg
header_permalink: http://www.metmuseum.org/collection/the-collection-online/search/337499
---
 
# {{ page.title }}

For my [Swiftacular](http://github.com/ccollicutt/swiftacular) project I'd like to figure out if I can run some automated tests and create a new Swiftacular cluster using libvirt. My Swiftacular configuration currently has five networks used to create an OpenStack Swift cluster (complete with a separate replication network). My Vagrantfile for Swiftacular sets up those networks, so I need to replicate that configuration using libvirt. 

By default with a libvirt host on Ubuntu 12.04 you get one network: the default network.

<pre>
<code># virsh net-list
Name                 State      Autostart
-----------------------------------------
default              active     yes       
</code>
</pre>     

That default network comes with a bridge and a dnsmasq server to provide dhcp addresses.

<pre>
<code># ifconfig | grep virbr
virbr0    Link encap:Ethernet  HWaddr 52:54:00:a3:05:65  
# ps ax  |grep dns
 4943 ?        S      0:02 /usr/sbin/dnsmasq -u libvirt-dnsmasq
SNIP!
</code>
</pre>

Each virtual machine booted using libvirt will have a default network configured. Below is a snippet of the xml defining a virtual machine. Note how the network to use is set to "default."

<pre>
<code>SNIP!
    <interface type='network'>
      <mac address='fa:16:3e:18:78:ae'/>
      <source network='default'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
SNIP!
</code>
</pre>

So first we need to add more networks, and then we need to configure virtual machines xml definition file with the networks, and then ensure that the vms has more interfaces set up.

## More libvirt networks

In Swiftacular I have five networks:

- eth0 - default - Used by Vagrant
- eth1 - public - 192.168.100.0/24 - The "public" network that users would connect to
- eth2 - lbssl - 10.0.10.0/24 - This is the network between the SSL terminator and the Swift Proxy
- eth3 - internal - 10.0.20.0/24 - The local Swift internal network
- eth4 - replication - 10.0.30.0/24 - The replication network which is a feature of OpenStack Swift starting with the Havana release

I'm going to replicate that with libvirt networks, and the eth0 network will be the default network.

Here is an xml definition file for the internal network I would like to add:

<pre>
<code><network>
  <name>internal</name>
  <forward mode='nat'/>
  <bridge name='internalbr0' stp='on' delay='0' />
  <ip address='10.0.20.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.0.20.2' end='10.0.20.254' />
    </dhcp>
  </ip>
</network>
</code>
</pre>

Now that the xml file has been created, we can use virsh net-define to define it.

<pre>
<code># virsh net-define internal_network.xml 
Network internal defined from internal_network.xml
</code>
</pre>

And we can see the network show up in virsh net-list.

<pre>
<code># virsh net-list --all
Name                 State      Autostart
-----------------------------------------
default              active     yes             
internal             inactive   no        
</code>
</pre>

But as can bee seen above, the network is not active, nor set to autostart. So let's do that.

<pre>
<code># virsh net-start internal
Network internal started

# virsh net-autostart internal
Network internal marked as autostarted

# virsh net-list
Name                 State      Autostart
-----------------------------------------
default              active     yes       
internal             active     yes            
</code>
</pre>

Also we have an internal bridge and a dnsmasq process for that network as well which sets up a leases file.

<pre>
<code># brctl show  |grep internal
internalbr0		8000.525400185a02	yes		internalbr0-nic

# ls /var/lib/libvirt/dnsmasq/internal.leases 
/var/lib/libvirt/dnsmasq/internal.leases
</code>
</pre>

I'm going to setup the rest of my networks in the same fashion.

<pre>
<code># virsh net-list 
Name                 State      Autostart
-----------------------------------------
default              active     yes       
internal             active     yes       
lbssl                active     yes       
private              active     yes       
replication          active     yes  
</code>
</pre>

Ok, now we can move onto the other steps.

## Add network interfaces to libvirt vm definition file

In order for those networks to be available to a virtual machine they need to be configured in the xml file that defines the vm.

<pre>
<code>SNIP!
        <interface type='network'>
            <source network='default'/>
                <model type='virtio'/>
        </interface>
        <interface type='network'>
            <source network='internal'/>
                <model type='virtio'/>
        </interface>
        <interface type='network'>
            <source network='lbssl'/>
                <model type='virtio'/>
        </interface>
        <interface type='network'>
            <source network='private'/>
                <model type='virtio'/>
        </interface>
        <interface type='network'>
            <source network='replication'/>
                <model type='virtio'/>
        </interface>
SNIP!
</code>
</pre>

That snippet would have to be in every vm definition file.

## Cloud-init and user-data

I use [cloud-init](http://cloudinit.readthedocs.org/en/latest/) to help configure vms in libvirt.

I have a user-data and meta-data file for each vm which is converted into an image file and connected to the virtual machine.

In my user-data file I configure the files that will setup eth2 to eth4 as dhcp. eth1 is already setup in the image, so I don't have to configure it with cloud-init.

<pre>
<code>SNIP!
write_files:
  - content: |
      auto eth1
      iface eth1 inet dhcp
    path: /etc/network/interfaces.d/eth1.cfg
  - content: |
      auto eth2
      iface eth2 inet dhcp
    path: /etc/network/interfaces.d/eth2.cfg
  - content: |
      auto eth3
      iface eth3 inet dhcp
    path: /etc/network/interfaces.d/eth3.cfg
  - content: |
      auto eth4
      iface eth4 inet dhcp
    path: /etc/network/interfaces.d/eth4.cfg
SNIP!
</code>
</pre>

When the vm boots with a cloud-init image attached, the cloud-init client on the vm will setup the files as configured so that each new interface on the new networks will get an IP from dnsmasq on the corresponding bridge.

Completely going over how cloud-init works is beyond the scope of this blog post, but it is a very important part of any cloud or virtualization platform. Ok, well not every virtualization system. OpenStack too though. :)

## Boot a vm

Let's boot a test vm.

I have a script called generic.sh that creates a custom cloud-init image and configures the image in the libvirt xml file for the vm.

I am using Ubuntu 14.04 as the OS for the virtual machine. This has a different cloud-init version that Ubuntu 12.04 so there may be differences in terms of cloud-init if trying to boot a Precise vm vs a Trusty vm.

<pre>
<code># ./generic.sh trusty testnetworks

Domain testnetworks defined from /tmp/tmp.yJ6w0CO3A2

Domain testnetworks started
</code>
</pre>

Now I have a vm called "testnetworks" running in libvirt. I can check the various dnsmasq lease files in /var/lib/libvirt/dnsmasq/*.leases to see if it got an IP address.

<pre>
<code># virsh list  |grep test
 40 testnetworks         running
# cat /var/lib/libvirt/dnsmasq/internal.leases 1398535697 52:54:00:72:db:0e 10.0.20.68 testnetworks *
</code>
</pre>

And we can see that it did.

Also we can find out the macs that were randomly given to the vm via libvirt using virsh dumpxml.

<pre>
<code># virsh dumpxml testnetworks | grep mac
    <type arch='x86_64' machine='pc-1.0'>hvm</type>
      <mac address='52:54:00:2d:52:8d'/>
      <mac address='52:54:00:72:db:0e'/>
      <mac address='52:54:00:93:1b:07'/>
      <mac address='52:54:00:76:61:ee'/>
      <mac address='52:54:00:72:79:48'/>
</code>
</pre>

If I ssh into the virtual machine I should be able to see all those interfaces and checkout the routing table. I can also get to the Internet via the default gateway.

<pre>
<code># ssh ubuntu@10.0.20.68
SNIP!
ubuntu@testnetworks:~$ ifconfig | grep eth
eth0      Link encap:Ethernet  HWaddr 52:54:00:2d:52:8d  
eth1      Link encap:Ethernet  HWaddr 52:54:00:72:db:0e  
eth2      Link encap:Ethernet  HWaddr 52:54:00:93:1b:07  
eth3      Link encap:Ethernet  HWaddr 52:54:00:76:61:ee  
eth4      Link encap:Ethernet  HWaddr 52:54:00:72:79:48 
ubuntu@testnetworks:~$ netstat -rn
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
0.0.0.0         192.168.122.1   0.0.0.0         UG        0 0          0 eth0
10.0.10.0       0.0.0.0         255.255.255.0   U         0 0          0 eth2
10.0.20.0       0.0.0.0         255.255.255.0   U         0 0          0 eth1
10.0.30.0       0.0.0.0         255.255.255.0   U         0 0          0 eth4
192.168.100.0   0.0.0.0         255.255.255.0   U         0 0          0 eth3
192.168.122.0   0.0.0.0         255.255.255.0   U         0 0          0 eth0
ubuntu@testnetworks:~$ ping -c 1 news.google.com
PING news.l.google.com (74.125.228.100) 56(84) bytes of data.
64 bytes from iad23s08-in-f4.1e100.net (74.125.228.100): icmp_seq=1 ttl=52 time=96.2 ms

--- news.l.google.com ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 96.265/96.265/96.265/0.000 ms
</code>
</pre>

So now I have five networks to play with!

Let's boot one more to see if we have connectivity between two vms.

<pre>
<code># ./generic.sh trusty testnetworks2
# cat /var/lib/libvirt/dnsmasq/internal.leases 
1398535915 52:54:00:84:c2:b9 10.0.20.27 testnetworks2 *
1398535697 52:54:00:72:db:0e 10.0.20.68 testnetworks *
</code>
</pre>

So we should be able to ping 10.0.20.27 from .68.

<pre>
<code>ubuntu@testnetworks:~$ ping -c 1 10.0.20.27
PING 10.0.20.27 (10.0.20.27) 56(84) bytes of data.
64 bytes from 10.0.20.27: icmp_seq=1 ttl=64 time=0.284 ms

--- 10.0.20.27 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.284/0.284/0.284/0.000 ms
</code>
</pre>

Yup.

So now we have five networks in libvirt, and virtual machines that can boot up and get a dhcp address on each of those networks. Hopefully this means I can work on automating testing of Swiftacular, probably by creating a custom inventory script.

Please let me know if you see any issues. One question I have is about the forward mode that should be set. I'm not sure it should be nat for my extra networks. Something to look into.






