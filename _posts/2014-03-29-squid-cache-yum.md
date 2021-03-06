---
layout: post
title: Use squid to cache RedHat/CentOS yum repositories
categories: 
---
 
# {{ page.title }}

Today I began working on adding RedHat/CentOS support to my Ansible based project [Swiftacular](https://github.com/ccollicutt/swiftacular) which deploys [OpenStack Swift](http://docs.openstack.org/developer/swift/).

Because, right now, it takes several Vagrant/Virtualbox virtual machines to run swiftacular, I like to make sure that I configure a local package caching server so that I don't kill my Internet connection by downloading the same package multiple times. 

When configuring Ubuntu vms I use apt-cacher-ng. But with RedHat I couldn't find, at least in a quick google search, a similar system for RedHat. (Though apparently apt-cacher-ng can cache rpms, but I haven't tried it with Redhat repos.)

I opted just to use squid. Surprisingly I couldn't find much for blog posts on using squid to proxy yum so I thought I'd post what I did, which is very simple, in case anyone else was trying to do the same thing. 

I imagine there are better ways to cache rpms locally, ie. on a laptop, but this is what I did. :)

## Setup squid

First, I installed squid on the package caching server.

<pre>
<code>cache# yum install squid
</code>
</pre>

The one change I have made to the squid configuration file is to set a cache_dir.

<pre>
<code>cache# grep cache_dir /etc/squid/squid.conf
#cache_dir ufs /var/spool/squid 100 16 256
cache_dir ufs /var/spool/squid 7000 16 256
</code>
</pre>

The first uncommented cache_dir is the default, which is not set unless uncommented. The second is my setting, which just has a larger maximum size for the cache directory. You could set that to whatever is appropriate for your environment.

You'll also have to allow connections to port 3128 via iptables, or just shut iptables down. (Normally I wouldn't shut iptables down, but this is just for a test system.)

Also note I started squid.

<pre>
<code>cache# netstat -ant | grep 3128
tcp        0      0 :::3128                     :::*                        LISTEN 
cache# service iptables stop
SNIP!
cache# service squid start
</code>
</pre>

So far so good.

## Configure the servers that will use the package cache

On all the servers that need to use the cache, I set the proxy configuration in their /etc/yum.conf file to be the cache server on port 3128.

<pre>
<code>server# head -12 yum.conf
[main]
cachedir=/var/cache/yum/$basearch/$releasever
keepcache=0
debuglevel=2
logfile=/var/log/yum.log
exactarch=1
obsoletes=1
gpgcheck=1
plugins=1
installonly_limit=5
proxy=http://192.168.100.20:3128
</code>
</pre>

Also, I commented out the mirrorlist lines in CentOS-Base.repo and uncommented the baseurl, eg.

<pre>
<code>[vagrant@swift-storage-02 yum.repos.d]$ grep "mirrorlist\|baseurl" CentOS-Base.repo 
# If the mirrorlist= does not work for you, as a fall back you can try the 
# remarked out baseurl= line instead.
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os
baseurl=http://mirror.centos.org/centos/$releasever/os/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates
baseurl=http://mirror.centos.org/centos/$releasever/updates/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras
baseurl=http://mirror.centos.org/centos/$releasever/extras/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=centosplus
baseurl=http://mirror.centos.org/centos/$releasever/centosplus/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=contrib
baseurl=http://mirror.centos.org/centos/$releasever/updates/$basearch/
</code>
</pre>

Normally you wouldn't want to do this, but because of the mirror list there will be many squid cache misses. So baseurl is better, and a closer mirror would be better still.

Next I removed the fastestmirror plugin:

<pre>
<code>server# rm -f /etc/yum/pluginconf.d/fastestmirror.conf 
</code>
</pre>

As an example, if I install php-common on one of the servers, then install it on another, it will hit the squid cache and just download it from there. So we only download the package once from the Internet, even if it's installed on several servers.

<pre>
<code>1396152104.396     10 192.168.100.202 TCP_HIT/200 537778 GET \
 http://mirror.centos.org/centos/6/updates/x86_64/Packages/php-common-5.3.3-27.el6_5.x86_64.rpm \
 - NONE/- application/x-rpm
</code>
</pre>

That's all it takes.

One thing I'm not sure about is the cache expiry time. I'm sure there are some other changes I will come across while using squid to cache yum repos, as I've only started using it in the last few hours, but I will ensure to come back and edit this post with new information.

If anyone has any comment, questions, concerns, or criticisms, do let me know. :)

## Ansible

PS. Everything I'm doing above is automated with [Ansible](http://ansible.com).

## Updates

- Added removing fastestmirror plugin
