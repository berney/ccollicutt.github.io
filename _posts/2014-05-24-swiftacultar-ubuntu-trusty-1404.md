---
layout: post
title: Swiftacular - Install OpenStack Swift on Ubuntu Trusty 14.04
categories: 
---

# {{ page.title }}

Recently I updated my [Swiftacular](https://github.com/ccollicutt/swiftacular) project to support Ubuntu Trusty 14.04 and OpenStack Icehouse (which Trusty comes with by default).

Swiftacular installs OpenStack Swift using Ansible on CentOS 6.5, Ubuntu 12.04, and Ubuntu 14.04. In CentOS and Ubuntu 12.04 it installs the OpenStack Havana release of Swift, and in Ubuntu 14.04 it uses Icehouse.

This post shows how to install OpenStack Swift Icehouse on Ubuntu 14.04 using Swiftacular.

## Requirements

Below are the tools required:

- Git
- Ansible 
- Virtualbox
- Vagrant
- Internet connection
- Enough resources for seven virtual machines

Note that I've only tested this on OSX Mavericks. But it should be quite easy to adapt to any situation if you don't mind changing some IP addresses in the group_vars/all file. Another goal I have is to have examples of getting Swiftacular up and running using libvirt + kvm, AWS, Digital Ocean, and other IaaS providers.

## Setup OpenStack Swift using Swiftacular

First, clone the Swiftacular repository and add some libraries. (I should put this all into a make file--it's on the todo list!)

<pre>
<code>curtis$ git clone https://github.com/ccollicutt/swiftacular.git
curtis$ cd swiftacular
curtis$ git clone https://github.com/openstack-ansible/openstack-ansible-modules library/openstack
</code>
</pre>

Edit the Vagrantfile so that the Ubuntu Trusty box will be used instead of the default Precise box.

<pre>
<code>curtis$ grep config.vm.box Vagrantfile 
    config.vm.box = "trusty64"
    config.vm.box_url = "https://cloud-images.ubuntu.com/vagrant/trusty/current/trusty-server-cloudimg-amd64-vagrant-disk1.box"
    #config.vm.box = "centos65"
    #config.vm.box_url = "http://puppet-vagrant-boxes.puppetlabs.com/centos-65-x64-virtualbox-nocm.box"
    #config.vm.box = "precise64"
    #config.vm.box_url = "http://files.vagrantup.com/precise64.box"
</code>
</pre>

Now we can use vagrant to start the virtual machines.

<pre>
<code>curtis$ vagrant up
SNIP!
   swift-storage-03: Warning: Remote connection disconnect. Retrying...
==> swift-storage-03: Machine booted and ready!
==> swift-storage-03: Checking for guest additions in VM...
==> swift-storage-03: Setting hostname...
==> swift-storage-03: Configuring and enabling network interfaces...
==> swift-storage-03: Mounting shared folders...
    swift-storage-03: /vagrant => /Users/curtis/working/swiftacular
curtis$ # done booting all the vms
</code>
</pre>

Test connectivity using ansible ping.

<pre>
<code>curtis$ ansible -m ping all
192.168.100.100 | success >> {
    "changed": false, 
    "ping": "pong"
}

192.168.100.50 | success >> {
    "changed": false, 
    "ping": "pong"
}

192.168.100.30 | success >> {
    "changed": false, 
    "ping": "pong"
}

192.168.100.200 | success >> {
    "changed": false, 
    "ping": "pong"
}

192.168.100.20 | success >> {
    "changed": false, 
    "ping": "pong"
}

192.168.100.201 | success >> {
    "changed": false, 
    "ping": "pong"
}

192.168.100.202 | success >> {
    "changed": false, 
    "ping": "pong"
}
</code>
</pre>

Good.

We can also use vagrant status to list all the vms. Their names should describe what they do fairly well. The lbssl server is the ssl termination point for the swift proxy. It's not doing any load balancing in this situation, but could if there were multiple proxy servers.

<pre>
<code>curtis$ vagrant status
SNIP!
swift-package-cache-01    running (virtualbox)
swift-keystone-01         running (virtualbox)
swift-lbssl-01            running (virtualbox)
swift-proxy-01            running (virtualbox)
swift-storage-01          running (virtualbox)
swift-storage-02          running (virtualbox)
swift-storage-03          running (virtualbox)
</code>
</pre>

Now lets set the ansible variables. The default variables should work, but the passwords are set to CHANGEME.

<pre>
<code>curtis$ cp group_vars/all.example group_vars/all
curtis$ # edit the group_vars/all file 
</code>
</pre>

Finally we can run ansible-playbook and install OpenStack Swift Icehouse across all the nodes. Depending on your internet connection and disk performance this should only take five or six minutes.

<pre>
<code>curtis$ ansible-playbook site.yml 

PLAY [package_cache] ********************************************************** 

GATHERING FACTS *************************************************************** 
ok: [192.168.100.20]
SNIP!
PLAY RECAP ******************************************************************** 
192.168.100.100            : ok=28   changed=22   unreachable=0    failed=0   
192.168.100.20             : ok=22   changed=16   unreachable=0    failed=0   
192.168.100.200            : ok=46   changed=37   unreachable=0    failed=0   
192.168.100.201            : ok=46   changed=37   unreachable=0    failed=0   
192.168.100.202            : ok=46   changed=37   unreachable=0    failed=0   
192.168.100.30             : ok=20   changed=17   unreachable=0    failed=0   
192.168.100.50             : ok=43   changed=36   unreachable=0    failed=0  
</code>
</pre>

If I ssh into the proxy server I can see what OpenStack packages have been installed.

<pre>
<code>curtis$ vagrant ssh swift-proxy-01
SNIP!
vagrant@swift-proxy-01:~$ dpkg --list | grep swift | tr -s " " | cut -f 2,3 -d " "
python-swift 1.13.1-0ubuntu1
python-swiftclient 1:2.0.3-0ubuntu1
swift 1.13.1-0ubuntu1
swift-object 1.13.1-0ubuntu1
swift-plugin-s3 1.7-3
swift-proxy 1.13.1-0ubuntu1
</code>
</pre>

Same with the storage servers.

<pre>
<code>vagrant@swift-storage-01:~$ dpkg --list | grep swift | tr -s " " | cut -f 2,3 -d " "
python-swift 1.13.1-0ubuntu1
python-swiftclient 1:2.0.3-0ubuntu1
swift 1.13.1-0ubuntu1
swift-account 1.13.1-0ubuntu1
swift-container 1.13.1-0ubuntu1
swift-object 1.13.1-0ubuntu1
</code>
</pre>

Now we can run a small test by uploading a text file into a container. I usually run this from the package cache server that is created as part of Swiftacular. But you can run the swift command line client from anywhere that the proxy server can be accessed from.

<pre>
<code>vagrant@swift-package-cache-01:~$ . testrc 
vagrant@swift-package-cache-01:~$ echo "swift is cool" > swift.txt
vagrant@swift-package-cache-01:~$ swift upload test_container swift.txt 
swift.txt
vagrant@swift-package-cache-01:~$ swift list
test_container
vagrant@swift-package-cache-01:~$ swift list test_container
swift.txt
</code>
</pre>

At this point we have a nice little working test cluster of OpenStack Swift Icehouse release running on Ubuntu Trusty 14.04.

There is still a lot of work I would like to do with Swiftacular. If you have any suggestions, comments, or criticisms please do let me know in the comments or enter an issue into the github repository.







