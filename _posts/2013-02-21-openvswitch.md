---
layout: post
title: Software defined networking, Openvswitch, and Ubuntu 12.04
categories:
---

# {{ page.title }}

Recently I've been testing over committing on KVM. Once I started running hundreds of virtual machines (vms) on a single node, I realized that in order to get them to do anything I have to access them over the network to run something like [Ansible](http://ansible.cc) playbooks designed to test load.

In order to provide networking resources to the vms, I decided to take a look at [Openvswitch](http://openvswitch.org) and what it takes to get it up and running with KVM and Ubuntu 12.04/Precise. 

## Software defined networking

Like cloud and big-data, [software defined networking](http://en.wikipedia.org/wiki/Software-defined_networking) (SDN) is a loaded term. But, like those terms, I feel I need to at least try to get a grasp of what it means.

> "A good working definition of SDN is the separation of the data and control functions of today's routers and other layer two networking infrastructure with a well-defined programming interface between the two." -- Via [Arstechnica](http://arstechnica.com/information-technology/2013/02/100gbps-and-beyond-what-lies-ahead-in-the-world-of-networking/2/)

SDN is a big part of [OpenStack](http://docs.openstack.org/trunk/openstack-network/admin/content/) as well. Starting with the [Folsom](http://www.openstack.org/software/folsom/) release, networking was split out into it's own _*as-a-Service_ capability called [Quantum](https://wiki.openstack.org/wiki/Quantum), whereas previously it was a sub-component of Nova. So given I'm a big fan, and user, of OpenStack, it's important for me to get a good grasp of SDN.

[Openflow](http://www.openflow.org/wk/index.php/OpenFlow_Tutorial) is also an important technology in SDN that requires some research time.

But, having said all that, basically I'm just going to install and use Openvswitch on a single compute node. :)

## Building on other's work

I followed these blog posts on configuring Openvswitch on Ubuntu 12.04/Precise:

- [How to build a SDN Lab without needing Openflow hardware](http://networkstatic.net/how-to-build-an-sdn-lab-without-needing-openflow-hardware/) 
- [Openvswitch and OpenFlow: Let's get started](http://en.community.dell.com/techcenter/networking/w/wiki/3820.openvswitch-openflow-lets-get-started.aspx)
- [Installing KVM and Openvswtich on Ubuntu](http://blog.scottlowe.org/2012/08/17/installing-kvm-and-open-vswitch-on-ubuntu/)

I'm not really doing anything new here--though I hope to at some point... :)

## Installation

I put together an [ansible playbook](https://github.com/ccollicutt/ansible_playbooks/blob/master/sdn/tasks/setup.yml) to install Openvswitch in Ubuntu 12.04. There is no easy, direct way (that I'm aware of) to install Openvswitch in Precise...unfortunately I just can't do @apt-get install openvswitch@ and have everything work like magic. I guess building the module is the only unusual thing, and this will disappear in future versions of Ubuntu--perhaps it's already not necessary in 12.10, not sure, haven't looked it up.

I'm not going to directly cut and paste my [ansible playbook](https://github.com/ccollicutt/ansible_playbooks/blob/master/sdn/tasks/setup.yml) into this post, but suffice it to say that most dependencies can be installed via `apt-get`, but there is one step required to build and module.

<pre>
<code># Install these packages:
#   - openvswitch-datapath-source 
#   - bridge-utils
#   - module-assistant  
#   - openvswitch-brcompat
#   - openvswitch-common
#   - openvswitch-switch
#   - linux-headers-3.2.0-23-generic
#   - linux-headers-generic-pae
# Then build the module:
$ module-assistant auto-install openvswitch-datapath
</code>
</pre>  

Next we set `BRCOMPAT=yes` in `/etc/default/openvswitch-switch` and restart `openvswitch-switch`.

## Pox

[Pox](http://www.noxrepo.org/pox/about-pox/), among other things, is an Openflow controller.

> "At its core, it’s a platform for the rapid development and prototyping of network control software using Python.  Meaning, at a very basic level, it’s one of a growing number of frameworks...for helping you write an OpenFlow controller." -- Via [Pox website](http://www.noxrepo.org/pox/about-pox/)

Hey, it's Python, it's Openflow...what else do I need. Sign me up. :)

<pre>
<code>$ cd /usr/local/src
$ git clone http://github.com/noxrepo/pox
$ zdaemon -p 'python /usr/local/src/pox/pox.py \
--no-cli forwarding.l2_learning' -d start
</code>
</pre>

And now Pox should be listening on port 6633:

<pre>
<code>$ netstat -ant | grep 6633
tcp        0      0 0.0.0.0:6633            0.0.0.0:*               LISTEN     
$ sudo lsof -i | grep 6633
python   34150   root    3u  IPv4 39211055      0t0  TCP *:6633 (LISTEN)
</code>
</pre>

More information about pox can be found on the [Openflow site](http://www.openflow.org/wk/index.php/OpenFlow_Tutorial#Controller_Choice:_POX_.28Python.29).

## Configure the bridge

Now we can add a bridge to the openv switch.

<pre>
<code>$ ovs-vsctl add-br br-int
</code>
</pre>

And then configure br-int (or whatever you've called the bridge), and I'm using the eth2 interface in this example.

<pre>
<code>$ ovs-vsctl add-port br-int eth2; ifconfig eth2 0; ifconfig br-int <IPv4 Address> \
netmask 255.255.255.0
</code>
</pre>

Let's tell Openvswitch to use the Pox controller that is running.

<pre>
<code>$ ovs-vsctl set-controller br-int tcp:<IPv4 Address>:6633
</code>
</pre>

Finally we need interface up and down scripts.

ifdown:

<pre>
<code>$ cat /sbin/ovs-ifdown
#!/bin/sh 
switch='br-int' 
/sbin/ifconfig $1 0.0.0.0 down 
ovs-vsctl del-port ${switch} $1
</code>
</pre>

ifup:

<pre>
<code>$ cat /sbin/ovs-ifup
#!/bin/sh switch='br-int' 
/sbin/ifconfig $1 0.0.0.0 up
 ovs-vsctl add-port ${switch} $1
 </code>
</pre>

Now we can boot some vms!

## Booting a vm

I'm booting vms via a script, and part of the `kvm` command line options is the network tap, which uses the ifup/ifdown scripts:

<pre>
<code>-net tap,script=/sbin/ovs-ifup,downscript=/sbin/ovs-ifdown
</code>
</pre>

And, here is a running instance:

<pre>
<code>$ ps ax | grep "kvm -drive" | head -1
  9831 ?        Sl    21:01 kvm -drive if=virtio,file=/mnt/intel/1.img -m 2048 \
  -boot a -net nic,macaddr=52:54:00:a1:c0:fd \
  -net tap,script=/sbin/ovs-ifup,downscript=/sbin/ovs-ifdown -nographic -vnc :1 \
   -chardev file,id=charserial0,path=/mnt/intel/1.console.log \
   -device isa-serial,chardev=charserial0,id=serial0 -chardev pty,id=charserial1 \
   -device isa-serial,chardev=charserial1,id=serial1 \
   -device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x5
</code>
</pre>

Right now there's about 300 vms running on this single compute node.

<pre>
<code>$ ps ax  |grep "kvm -drive" | wc -l
301
</code>
</pre>

Fun stuff. :)

## What now?

Well, I've achieved my goal of getting Openvswitch up and running to enable networking between vms on a single compute node. 

<pre>
<code>ubuntu@ubuntu:~$ ifconfig eth0 | grep 192
          inet addr:192.168.100.111  Bcast:192.168.100.255  Mask:255.255.255.0
ubuntu@ubuntu:~$ ping -c 1 192.168.100.23
PING 192.168.100.23 (192.168.100.23) 56(84) bytes of data.
64 bytes from 192.168.100.23: icmp_req=1 ttl=64 time=0.452 ms

--- 192.168.100.23 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.452/0.452/0.452/0.000 ms
</code>
</pre>

I think my next step will be to work in multiple virtual machines to see if I can do some of the interesting and useful things that Openflow is capable of, and to find out how I can work with the Pox system to learn more about SDN.

Another important thing to do is to get a test environment of OpenStack Folsom (or Grizzly) up and running to see how Quantum utilizes Openflow and SDN.

I'm hoping to spend the next few days learning about [Pox](http://www.noxrepo.org/pox/about-pox/). Wish me luck. :)
