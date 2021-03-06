---
layout: post
title: My OpenBSD Lab
categories:
header_image: https://github.com/ccollicutt/ccollicutt.github.com/raw/master/img/openbsd_lab.jpg
---

# {{ page.title }}

Above you can see my little OpenBSD lab. For some reason my workplace has several different kinds of small form factor servers, such as the big silver Netcom box, a couple of [Soekris](http://soekris.com) boxes, and I can't even tell what make/model the little blue boxes are. They are all essentially small, fanless servers with at least two ethernet ports. Since they weren't being used for anything and I needed to setup a few test installs of OpenBSD, I went to work. :)

The "lab" is comprised of two [carped](http://www.openbsd.org/faq/pf/carp.html) firewalls (the blue boxes; note the red cable between them is a cross over cable for pfsync traffic), a couple of cheap desktop switches, and the square silver thing is an OpenBSD bridge that goes in between the two switches. The carped firewalls are connected to another part of my test network, which eventually leads out to the Internet. I also have a couple of OpenBSD virtual machines running on my RHEL 5 xen dom0. One of them provides an OpenBSD pxe boot solution to install OpenBSD onto systems like the Soekris that can't boot from USB. 

When testing I plug my laptop into the internal switch (the green cable), so that in order to get out to the Internet it has to go over the bridge and through the firewalls. "Over the Bridge and through the Firewall"...sounds like a book Hemingway would have written if he was a Unix security administrator. ;)

Using this little lab I can test out all kinds of interesting OpenBSD functionality such as packet filtering with pf, virtual IP address failover with carp, bridging, and network authentication using authpf, along with anything else that I need to work on or try out in the OpenBSD world, or just try to stay familiar with OpenBSD in general.

Bridge up!

<pre>
<code>$ ifconfig bridge0 up
</code>
</pre>







