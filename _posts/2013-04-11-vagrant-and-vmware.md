---
layout: post
title: Vagrant and vmware
categories:
---

# {{ page.title }}

After IRC, [vagrant](http://vagrantup.com) is probably my most important development tool, mostly because I like to use and investigate openstack, which means using a lot of virtual machines.

Recently Hashicorp released [Vagrant 1.1](http://www.hashicorp.com/blog) which introduces the idea of [providers](http://docs.vagrantup.com/v2/providers/index.html). Previously vagrant only supported virtualbox, but now, with 1.1, plugins can be written to support almost any virtualization system that has a command line or API interface of some sort.

For example:

- [VMWare Fusion](http://docs.vagrantup.com/v2/vmware-fusion/index.html) (note that this is a paid plugin)
- [AWS](http://gigaom.com/2013/02/13/developers-rejoice-vagrant-finds-a-home-in-the-amazon-cloud/) ([github repo](https://github.com/mitchellh/vagrant-aws))
- [RackSpace](https://github.com/mitchellh/vagrant-rackspace)
- [OpenStack](https://github.com/cloudbau/vagrant-openstack) (based on the rackspace plugin)

The provider I'm going to focus on here is vmware fusion.

## vmware_fusion

One of the things I've learned about using brand new technologies is that they often don't work and/or don't have any documentation, which frankly are about the same thing to me. That sounds like a kind of grumpy thing to say, but I'm kind of grumpy today. :)

Regardless, I went ahead and bought vmware fusion (which is cheap, BTW, at $49) and also the [vagrant vmware_fusion plugin](http://www.vagrantup.com/vmware) (which is $79).

I think this is the first time that I've encountered a plugin that was more expensive than the actual application it was plugging into, but I can understand the pricing because Fusion is probably under-valued, or at least under-priced. Plus the $79 goes towards the development of vagrant, which I use _a lot_.

Recently I deployed [packstack](http://serverascode.com/2013/03/13/first-look-packstack.html) via vagrant and *virtualbox*, and I wanted to do the same with vmware_fusion, but I ran into a few problems, which I'm going to spend the rest of the post detailing.

_NOTE: I should say that nothing here is the vmware_fusion plugins fault. I'm not blaming the plugin at all. Rather just detailing some of the pain points I've encountered, which will no doubt disappear as more people use vmware fusion and vagrant together, and as I get my act together. I'll try to update this post as I find out new information. :)_

## Routes collide!

I have both vmware fusion and virtualbox installed on my macbook retina. Unfortunately, virtualbox has an iron grip on its networks.

> VirtualBox hangs on to its network devices ("vboxnet") for dear life. I haven't figured out yet how to actually get rid of them except restarting your computer. -- [Mitchell Hashimoto](https://groups.google.com/d/msg/vagrant-up/DKxnHU4_aOg/68JzFjJ-14sJ)

If you encounter the below error, either change subnets (perhaps in virtualbox, perhaps in the vagrantfile, not sure) or reboot.

<pre>
<code>$ vagrant up apis --provider=vmware_fusion
Bringing machine 'apis' up with 'vmware_fusion' provider...

[apis] Verifying vmnet devices are healthy...
The VMware network device 'vmnet2' can't be started because
its routes collide with another device: 'vboxnet'. Please

either fix the settings of the VMware network device or stop the
colliding device. Your machine can't be started while VMware
networking is broken.
</code>
</pre>

Again, not vmware_fusion's fault, but still a pain. I can't simply un-install virtualbox...yet.

## vmx settings

Often we want to change the settings in the virtual machine, settings such as memory, number of cpus, etc.

Unfortunately vmx is an undocumented format.

> VMX is an undocumented format. You'll have to google, unfortunately. :) -- [Mitchell Hashimoto](https://groups.google.com/d/msg/vagrant-up/DKxnHU4_aOg/68JzFjJ-14sJ)

But at the very least here is how to set memory:

<pre>
<code>config.vm.provider :vmware_fusion do |p|
  p.vmx['memsize'] = '2048'
end
</code>
</pre>

As more people use vmware_fusion there will be better documentation on vmx settings.

## Centos6 box

While Hashicorp has conveniently provided a [base precise64](http://files.vagrantup.com/precise64.box) box for vagrant, there isn't an official centos box. I have previously tried to create a centos6 box for vagrant, but haven't had much luck, and that was with vagrant < 1.1 and there is even less documentation on the process now.

Then I noticed that [vagrantbox.es](http://www.vagrantbox.es/) (which is a very handy site!) has a centos6 box for vmware_fusion, so I grabbed that:

- [CentOS 6.4 x86_64 Minimal VMware Fusion (VMware Tools, Chef 11.4.0, Puppet 3.1.1)](https://dl.dropbox.com/u/5721940/vagrant-boxes/vagrant-centos-6.4-x86_64-vmware_fusion.box)

Unfortunately it doesn't seem to work when multiple interfaces are specified in the vagrantfile, so that doesn't help me much on my quest to run packstack in vmware_fusion. If anyone knows of a good centos6 box, or notices that I'm doing something wrong, please let me know!

Here's the networking part of the config:

<pre>
<code>config.vm.network :private_network, ip: "172.10.0.200", :netmask => "255.255.0.0"
config.vm.network :private_network, ip: "10.10.0.200", :netmask => "255.255.0.0" 
</code>
</pre>

Let's boot it:

<pre>
<code>$ vagrant up --provider=vmware_fusion
Bringing machine 'default' up with 'vmware_fusion' provider...
[default] Cloning Fusion VM: 'centos65fusion'. This can take some time...
[default] Verifying vmnet devices are healthy...
[default] Preparing network adapters...
[default] Starting the VMware VM...
[default] Waiting for the VM to finish booting...
[default] The machine is booted and ready!
[default] Forwarding ports...
[default] -- 22 => 2222
[default] Configuring network adapters within the VM...
The following SSH command responded with a non-zero exit status.
Vagrant assumes that this means the command failed!

/sbin/ifup eth1 2> /dev/null
</code>
</pre>

Ooops, shouldn't be seeing the failed command.

What's the networking like?

<pre>
<code>$ vagrant ssh
[vagrant@vagrant-centos-6 ~]$ ifconfig
eth0      Link encap:Ethernet  HWaddr 00:0C:29:24:6A:AD  
          inet addr:192.168.134.146  Bcast:192.168.134.255  Mask:255.255.255.0
          inet6 addr: fe80::20c:29ff:fe24:6aad/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:432 errors:426 dropped:0 overruns:0 frame:0
          TX packets:293 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:48690 (47.5 KiB)  TX bytes:38046 (37.1 KiB)
          Interrupt:19 Base address:0x2024 

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 b)  TX bytes:0 (0.0 b)

</code>
</pre>

Nope that's not what I wanted at all.

Ok, now let's use the exact same vagrantfile but with the offical vmware_fusion ubuntu box.

<pre>
<code>config.vm.box = "precise64"
</code>
</pre>

Vagrant up!

<pre>
<code>#
# Destroy the old one
# 

$ vagrant destroy
[default] Stopping the VMware VM...
[default] Deleting the VM...

#
# Edit the vagrantfile to use precise64 basebox
#

$ vi Vagrantfile

#
# Boot it
# 

$ vagrant up --provider=vmware_fusion
Bringing machine 'default' up with 'vmware_fusion' provider...
[default] Cloning Fusion VM: 'precise64'. This can take some time...
[default] Verifying vmnet devices are healthy...
[default] Preparing network adapters...
[default] Starting the VMware VM...
[default] Waiting for the VM to finish booting...
[default] The machine is booted and ready!
[default] Forwarding ports...
[default] -- 22 => 2222
[default] Configuring network adapters within the VM...
[default] Enabling and configuring shared folders...
[default] -- vagrant-root: /Users/curtis/working/vagrant/grizzly

#
# SSH into the box...
# 

$ vagrant ssh
Welcome to Ubuntu 12.04.1 LTS (GNU/Linux 3.2.0-29-virtual x86_64)

 * Documentation:  https://help.ubuntu.com/
Last login: Thu Jan 31 13:48:53 2013
vagrant@precise64:~$ ifconfig
eth0      Link encap:Ethernet  HWaddr 00:0c:29:29:dc:aa  
          inet addr:192.168.134.139  Bcast:192.168.134.255  Mask:255.255.255.0
          inet6 addr: fe80::20c:29ff:fe29:dcaa/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:375 errors:367 dropped:0 overruns:0 frame:0
          TX packets:241 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:41473 (41.4 KB)  TX bytes:33220 (33.2 KB)
          Interrupt:18 Base address:0x2024 

eth1      Link encap:Ethernet  HWaddr 00:0c:29:29:dc:b4  
          inet addr:172.10.0.200  Bcast:172.10.255.255  Mask:255.255.0.0
          inet6 addr: fe80::20c:29ff:fe29:dcb4/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1 errors:1 dropped:0 overruns:0 frame:0
          TX packets:6 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:209 (209.0 B)  TX bytes:468 (468.0 B)
          Interrupt:16 Base address:0x20a4 

eth2      Link encap:Ethernet  HWaddr 00:0c:29:29:dc:be  
          inet addr:10.10.0.200  Bcast:10.10.255.255  Mask:255.255.0.0
          inet6 addr: fe80::20c:29ff:fe29:dcbe/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1 errors:1 dropped:0 overruns:0 frame:0
          TX packets:6 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:209 (209.0 B)  TX bytes:468 (468.0 B)
          Interrupt:17 Base address:0x2424 

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
</code>
</pre>

That's what I expect to see.

## Example vagrantfiles

_UPDATE (April 18th, 2013): This Vagrantfile now doesn't seem to work with Vagrant 1.2_

I have been searching for good examples of vagrantfiles that use vmware_fusion.

So far I've just found this one:

- [https://github.com/uksysadmin/OpenStackCookBook-1/blob/master/Vagrantfile](https://github.com/uksysadmin/OpenStackCookBook-1/blob/master/Vagrantfile)

But I will keep an eye out for other examples.

## UPDATE (April 18th, 2013): New network problem

This is a new one...now this can't be virtualbox's fault. 

<pre>
<code>$ vagrant up --provider=vmware_fusion
Bringing machine 'percona0' up with 'vmware_fusion' provider...
Bringing machine 'percona1' up with 'vmware_fusion' provider...
Bringing machine 'percona2' up with 'vmware_fusion' provider...
Bringing machine 'haproxy0' up with 'vmware_fusion' provider...
Bringing machine 'haproxy1' up with 'vmware_fusion' provider...
[percona0] Cloning Fusion VM: 'precise64'. This can take some time...
[percona0] Verifying vmnet devices are healthy...
The VMware network device 'vmnet1' can't be started because
its routes collide with another device: 'vmnet13'. Please
either fix the settings of the VMware network device or stop the
colliding device. Your machine can't be started while VMware
networking is broken.
</code>
</pre>

Instructions from Mitchell, edit _/Library/Preferences/VMware\ Fusion/networking_ and:

bq.. Get rid of all the lines in that file _except_ the ones that start with "answer VNET_1_" or "answer VNET_8_". We want to keep those, as they're the default networks that ship with Fusion. After that, open VMware Fusion.app, then run these commands in a separate terminal:

sudo /Applications/VMware\ Fusion.app/Contents/Library/vmnet-cli --stop
sudo /Applications/VMware\ Fusion.app/Contents/Library/vmnet-cli --configure
sudo /Applications/VMware\ Fusion.app/Contents/Library/vmnet-cli --start

Then run

sudo /Applications/VMware\ Fusion.app/Contents/Library/vmnet-cli --status

And tell me the output. Should only have the vmnet1/vmnet8 devices. After
THAT you shoudl be good to go again.

VMware networking is an absolute nightmare. - [Mitchell](https://mail.google.com/mail/u/2/?ui=2&ik=7b53664106&view=om&th=13e1f0d872dd857b)

It sucks that it's an edge case, but I still hope that there is some code added to help in situations like this. Thanks to Mitchell for responding on the mailing list, as now I can continue on with other problems. :)


## Conclusion

I love vagrant, but am having a heck of a time with vmware_fusion and centos6. Having said that, I *KNOW* that things are going to get better as I learn and as more people start using vagrant and vmware_fusion.

Hats off to Mitchell for creating a great development tool, one that I use every day!
