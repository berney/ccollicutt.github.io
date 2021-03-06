---
layout: post
title: Swiftacular - deploy OpenStack Swift with Ansible on CentOS
categories: 
header_image: /img/met/DT1540.jpg
header_permalink: http://www.metmuseum.org/collection/the-collection-online/search/10482 
---
 
# {{ page.title }}

[Swiftacular](https://github.com/ccollicutt/swiftacular) is a project that can deploy an OpenStack Swift cluster using [Ansible](http://ansible.com). In this post I'll talk about a recent feature I added which is the ability to deploy to RedHat/CentOS 6.

## Providers

For the most part Swiftacular is used for getting used to how OpenStack [Swift](http://docs.openstack.org/developer/swift/) works and is installed, and will usually be deployed using Vagrant and Virtualbox. 

One of the features I hope to add is the ability to use multiple provisioners, such as IaaS providers like as Digital Ocean, OpenStack, etc. Having said that, it's not required to use Vagrant, you could easily change some IPs around in the ansible hosts file, make a couple changes in other spots, and deploy to any servers, whether they are provided by Vagrant, virtual servers or even bare metal.

But for now, Swiftacular uses Vagrant and Virtualbox.

## Deploying to CentOS

By default the Vagrantfile that comes with Swiftacular will use the Ubuntu 12.04 Precise 64bit box that the Vagrant project provides.

But, if you would like to try deploying OpenStack Swift to CentOS 6 Swiftacular supports that as well, and doing so is as easy as changing which box the Vagrantfile points to.

<pre>
<code>#config.vm.box = "centos65"
#config.vm.box_url = "http://puppet-vagrant-boxes.puppetlabs.com/centos-65-x64-virtualbox-nocm.box"
config.vm.box = "precise64"
config.vm.box_url = "http://files.vagrantup.com/precise64.box"
</code>
</pre>

So if you would like to deploy to CentOS, simply make the Vagrantfile look like this:

<pre>
<code>config.vm.box = "centos65"
config.vm.box_url = "http://puppet-vagrant-boxes.puppetlabs.com/centos-65-x64-virtualbox-nocm.box"
#config.vm.box = "precise64"
#config.vm.box_url = "http://files.vagrantup.com/precise64.box"
</code>
</pre>

Then, once vagrant up is run and the virtual machines are created, the site.yml playbook can be run and OpenStack Swift will be deployed using the [RDO packages](http://openstack.redhat.com/Main_Page). The playbooks will detect that it is a RedHat-like operating system and deploy the right packages and files for that operating system.

## About RedHat RDO

Apparently RDO is a meaningless acronym, but I tend to think of it as "RedHat's Distribution of OpenStack."

One thing I found while using RDO is that the rdo-release.repo has a priorities setting. Note the "priority=98" option below.

<pre>
<code>[vagrant@swift-proxy-01 yum.repos.d]$ cat rdo-release.repo 
[openstack-havana]
name=OpenStack Havana Repository
baseurl=http://repos.fedorapeople.org/repos/openstack/openstack-havana/epel-6/
enabled=1
skip_if_unavailable=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-RDO-Havana
priority=98
</code>
</pre>

This means that it will be used before other repos that don't have a priority setting (as I believe the default is 99, and the lower the number the higher the priority). 

But, this requires the yum priorities plugin to work.

<pre>
<code>[vagrant@swift-proxy-01 yum.repos.d]$ rpm -qa | grep priorities
yum-plugin-priorities-1.1.30-17.el6_5.noarch
</code>
</pre>

Thus that plugin must be installed in order to use the priority setting, otherwise the wrong packages may be installed.

## Run Swiftacular

Once the Vagrantfile has been changed to use a CentOS 6 box and vagrant up has been run, swiftacular can install the Swift cluster.

Below I show that the servers are all CentOS 6.5.

<pre>
<code>curtis$ ansible -a "cat /etc/redhat-release" all
192.168.100.20 | success | rc=0 >>
CentOS release 6.5 (Final)

192.168.100.30 | success | rc=0 >>
CentOS release 6.5 (Final)

192.168.100.200 | success | rc=0 >>
CentOS release 6.5 (Final)

192.168.100.100 | success | rc=0 >>
CentOS release 6.5 (Final)

192.168.100.50 | success | rc=0 >>
CentOS release 6.5 (Final)

192.168.100.202 | success | rc=0 >>
CentOS release 6.5 (Final)

192.168.100.201 | success | rc=0 >>
CentOS release 6.5 (Final)
</code>
</pre>

Now to run the site playbook. Note that occasionally I've had to run the site playbook twice because of a [bug](https://github.com/ccollicutt/swiftacular/issues/12). So below is on the second run.

<pre>
<code>curtis$ #second run
curtis$ ansible-playbook site.yml 
SNIP!
PLAY RECAP ******************************************************************** 
192.168.100.100            : ok=26   changed=1    unreachable=0    failed=0   
192.168.100.20             : ok=22   changed=0    unreachable=0    failed=0   
192.168.100.200            : ok=42   changed=3    unreachable=0    failed=0   
192.168.100.201            : ok=42   changed=3    unreachable=0    failed=0   
192.168.100.202            : ok=45   changed=8    unreachable=0    failed=0   
192.168.100.30             : ok=17   changed=2    unreachable=0    failed=0   
192.168.100.50             : ok=35   changed=0    unreachable=0    failed=0   
</code>
</pre>

Once the Swiftacular playbook has completed successfully, swift is up and running.

<pre>
<code>curtis$ vagrant ssh swift-package-cache-01
working on swift-package-cache-01 with ip of 192.168.100.20
Last login: Sat Apr 12 09:26:12 2014 from 192.168.100.1
Welcome to your Vagrant-built virtual machine.
[vagrant@swift-package-cache-01 ~]$ . testrc 
[vagrant@swift-package-cache-01 ~]$ swift list
[vagrant@swift-package-cache-01 ~]$ #nothing there yet, let's add 100 files
[vagrant@swift-package-cache-01 ~]$ mkdir test
[vagrant@swift-package-cache-01 ~]$ for i in $(seq 1 100); do echo "swift $i" > \
test/swift$i.txt; done
[vagrant@swift-package-cache-01 ~]$ swift upload test test
test/swift76.txt
test/swift28.txt
test/swift79.txt
test/swift27.txt
SNIP!
[vagrant@swift-package-cache-01 ~]$ swift list
test
[vagrant@swift-package-cache-01 ~]$ swift list test | head
test/swift1.txt
test/swift10.txt
test/swift100.txt
test/swift11.txt
test/swift12.txt
test/swift13.txt
test/swift14.txt
test/swift15.txt
test/swift16.txt
test/swift17.txt
[vagrant@swift-package-cache-01 ~]$ swift list test  |wc -l
100
</code>
</pre>

Now that I've uploaded 100 files, and we have replicas set to 2, I can do a little digging around to see where files ended up.

<pre>
<code>curtis$ ansible -m shell -a "sudo find /srv | grep data | wc -l" storage
192.168.100.202 | success | rc=0 >>
0

192.168.100.200 | success | rc=0 >>
100

192.168.100.201 | success | rc=0 >>
100
</code>
</pre>

Above it can be seen that there are 100 files on two of the three servers, which makes sense if we want two copies of each file. Though I still have a lot to learn about what OpenStack Swift is actually doing in the background.

## Conclusion

Swiftacular now has the basic ability to deploy OpenStack Swift to RedHat 6 based operating systems using the RDO packages. There is a lot more to be done, but now with Ubuntu 12.04 and RedHat 6 support I can move onto adding other interesting features. 

If you have any suggestions or run into errors, please do let me know and I'll fix them. I'd love to get some testers. :)



