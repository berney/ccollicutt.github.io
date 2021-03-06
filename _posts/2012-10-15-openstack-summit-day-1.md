---
layout: post
title: OpenStack 2012 Summit Day &#35;1
categories:
---

# {{ page.title }}

This week I'm at the OpenStack 2012 Summit. It's by far the biggest conference I've been to. Usually I go to conferences on security, such as [H.O.P.E](http://en.wikipedia.org/wiki/Hackers_on_Planet_Earth) or [CanSec](http://cansecwest.com/) or even smaller library related conferences. Given that OpenStack is one of the fastest growing open source projects of all time, it's no surprise to find out that the conference has grown from 75 people a couple years ago, to about 1300 this year, up from 700 only a year ago. It's massive and it's growing.

## tl;dr

- Need to get into Ceph
- Database as a service too
- Upgrading is a big issue in OpenStack 
- San Diego is nice 
- OpenStack is kinda a big deal

## Opening "keynote"

Not much to say here, some summit housekeeping with trippy techno rave music to wake me up. :)

## Open Compute

The first session I attended was by [Cole Crawford](https://twitter.com/coleinthecloud) of the [Open Compute Foundation](http://opencompute.org/). Basically this was an overview of the history of things like the 19" rack (it came from waaaaay back from the train system) and how we need to change our standards to allow things like interoperability and other good stuff.

I was hoping to find out how a small organization--like the one that I work for--could be involved in Open Compute, specifically how we could access similar hardware as Facebook and others are using. I didn't quite get that out of the presentation, but hopefully in the future I will be able to get us into Open Compute hardware in some fashion or another.

At the very least I signed up for the mailing list. :)

## Intercloud Object Storage: Colony

The [Colony](https://github.com/nii-cloud/colony) session was lead by Shigetoshi Yokoyama of the [Japan National Institute of Informatics](http://www.nii.ac.jp/en/) and there is a copy of the slides  [online](http://www.slideshare.net/shigetoshi-yokoyama/openstack-design-summit-colony-session-12603753). Colony is described as "federated swifts" for intercloud object storage services. 

Having worked in a world class library that was interested in running petabytes of storage, I'm enthralled with object storage. It's an important storage paradigm, and surprisingly, for reasons unknown, one that doesn't seem to get a lot of attention in Canada. Perhaps it's because startups and other organizations just use S3 and haven't run into any privacy issues as of yet. Speaking of Amazon S3: it has a trillion+ objects stored now.

I think that in situations where we need to store a lot of replicated data--such as what a library or researcher would like to, or rather *should* like to store--object storage such as swift is a great way to go. (Though that said, perhaps systems like [Amazon's Glacier](http://aws.amazon.com/glacier/) make more sense for those use cases, but maybe not.) I was at a presentation by the then [CIO of the University of Alberta](http://webdocs.cs.ualberta.ca/~jonathan/) who figured there was three to five petabytes of research data on campus that needs to be preserved. That's a lot of storage, especially when factoring in replication, and it really needs to happen at some point--and it can't just be a huge tape system.

Further, considering that libraries and researchers  (ie. their respective Universities and organizations), are supposed to work together it would make sense to be able to have some kind of interoperability between object storage clouds, private or otherwise, even if it's just for some geographic separation.

Certainly Colonly is going to be a project to keep an eye on as object storage evolves and we try to do it across data centers and organizations.

## Operating your OpenStack Private Cloud

The next session I attended was [Operating your OpenStack Private Cloud](http://www.openstack.org/summit/san-diego-2012/openstack-summit-sessions/), which is good because that is one of the things that I do. I manage a small, eight node, OpenStack cluster that will eventually backend a [Apache VCL](https://cwiki.apache.org/VCL/apache-vcl.html) setup for virtual classrooms. So a very small private cloud, but I've still hit a few "pain points" that larger installations do.

A few points the speaker made:

- Monitoring: statsd, graphite, etc
- Operations tools don't exist for OpenStack (yet)--we need better ops tools so he wrote [Pulsar](https://github.com/JCallicoat/pulsar)
- Would be nice to see an operations dashboard (perhaps in horizon)
- Eg. can't get a list of all instances on a node and their IP addresses
- Some tools need hostnames and some need ids...why
- DSH for Ubuntu, PSH for Redhat?
- Don't forget your bashfu; still working with Linux :)
- Database backups
- [Holland](http://hollandbackup.org/)
- Performance and scale considerations
- Block storage solution of some kind (cinder, or cinder + some vendor)
- Local disk - raw image is only slightly faster than qcow2, but qcow2 may make better business sense
- IO will degrade on local disks when glance copies images between machines
- Scheduling
- CFQ made the most sense
- You have the power to change all this! You have the power to change your scheduler! Benchmark workloads and plan accordingly.
- Lots of things you can do with glance performance-wise
- Image caching
- If you're using qcow2 it doesn't have to move the whole image (something to check into)
- Don't want swift to become unbalanced
- They use chef for automated deployment
- controller in 15 minutes
- compute in two
- Day to day tasks
- Made up of mostly dealing with new issues
- Eg. resizing
- Hardware failures...still have all this hardware
- Don't forget--it's not (always) you, it's a bug
- Nice thing about a private cloud is you can pick the architecture that matches your requirements
- OpenStack/We need to provide an upgrade path (I hear this 20 times today)
- Metrics
- Load average is actually quite telling
- How long does an API call take?

## Lunch!

Lunch was Ok. There are so many people here it's tough to move around. And given OpenStack gave out giant bags, everyone (like me) has their own personal giant bag, plus the one they got from OpenStack (though I gave mine back). Giant bags galore!

Then I had a beer at the bar downstairs:

![](https://raw.github.com/ccollicutt/ccollicutt.github.com/master/img/openstack_summit_2012_bar.jpg)

## Database as a Service


This session is specifically about [RedDwarf](http://wiki.openstack.org/DatabaseAsAService). RedDwarf is taking MySQL and treating it like a "first class citizen" in OpenStack. It's a managed MySQL database service.

I believe that SQL databases should be separated out from compute. That said, there are many people that would disagree with me. There's a theory out there that storage and compute should be together on the same nodes, and if that's the case then I assume so would database. Right now I would prefer to separate out storage, compute, and database. Maybe it doesn't fit perfectly with the cloud paradigm, but sometimes you have to apply technology not just theorize about it. SQL isn't going anywhere, and neither is its unique workload.

HP and Rackspace have slightly different implementations of RedDwarf, and both of their services are in production. Each are trying to keep their system compatible with OpenStack, and have committed themselves to using the same CLI: [python-reddwarfclient](https://github.com/hub-cap/python-reddwarfclient). Further, they are both using OpenStack-common and Swift for securely storing database snapshots.

The team has a desire to bring RedDwarf forward, and they want to involve more community usage and get it into incubation in OpenStack and at that point a lot of doors open, such as a GUI in Horizon.

You can get a test install using the below commands, and it will "fake" a nova backend.

<pre>
<code>$ git clone https://github.com/hub-cap/reddwarf_lite
$ ./reddwarf_lite/bin/start_server.sh
</code>
</pre>

There was also a lot of discussion as to the fact that RedDwarf could be used to build _Other Things as a Service_ and also the fact that they are running it on top of OpenVZ versus another virtualization hypervisor and did so for performance reasons (up to 30% better was a number mentioned).

## Extending OpenStack for Fun and Profit: Creating New Functionality with the Enhancements API

This session is about having two sides--one side is OpenStack, and the other side is "your innovation". It's about getting OpenStack and your innovation to talk to one another; about getting your innovation to market and make money.

The speaker, Tim Smith, is the [co-founder](http://www.gridcentric.com/company/management/) of GridCentric which has essentially implemented _fork()_ for virtual machines. The main idea is spin up replica instances quickly and efficiently. They are reinventing boot, and in order to do that they need to extend Nova. GridCentric's code for their extension is up on [github](https://github.com/gridcentric/openstack) and would be a good example to work from. (I've looked at GridCentric's offerings before in relation to virtual classrooms--ie. not having a bootstorm when a bunch of students startup instances for a class. With something like GridCentric's technology you don't have to boot the instance.)

Nova is a framework providing:
- A standardized API
- Messaging/RPC
- Database-backed ORM

Nova can be extended by:
- API extensions
- Custom "services"
- Nova CLI extensions
- Dashboard extensions

Tim suggests that there are a few business challenges such as the fast evolution of OpenStack but generally seems positive about extending OpenStack. Certainly GridCentric has based some of their business on OpenStack.

A commenter noted that there are actually pre-action and post-action boot hooks in Folsom which may help to extend Nova.

## Storing VMs with Cinder and Ceph RBD

Considering how interested I am in storage, this project, [Ceph](http://ceph.com/), is one of the most important I know of. So when Josh from Inktank talks about Ceph, I'm here to listen. As a note, this room is standing room only for this session!

Ceph is designed for scalability--it has no single point of failure, and you don't have to be locked into a single hardware vendor. It's software based and self-managing. There is also the ability to run custom functions on the storage node--such as create a thumbnail from an image file.

[CRUSH](http://ceph.com/wiki/Custom_data_placement_with_CRUSH) is basically what sets Ceph apart from other storage systems. It's a pseudo-random placement algorithm and it ensures even distribution across the cluster and has a rules-based system. Basically it avoids look up tables, which eventually kill distributed storage. Further, unlike RAID the entire cluster can recover in parallel.

RBD, the Rados Block Device, is the part of Ceph used to access block storage. KVM has a native driver for RBD. They are thin-provisioned, striped across nodes, and support snapshots and cloning. With RBD you can spin up thousands of virtual machines with the same base image.

Why use block storage in OpenStack?
- Persistent
- Not tied to a single host--decouple storage from compute
- Enables live migration

Now with Folsom, Cinder has learned how to talk to Glance, which can respond with Images--ie. boot from volume. But still involves a large copy from Glance. However, images can be stored in RBD, and have been able to for some time! So we can skip the copy and just use RBD directly.

That said, it isn't quite perfect yet. There are enhancements being made in Grizzly. Eg. is not in Horizon, but is in the API and CLI.

NOTES: 
- The Ceph file system is not recommended for production, but everything else is, such as RBD. If you don't use the Ceph file system you don't need metadata servers.
- According to Josh, in terms of running OSDs on the same node as the hypervisor, as long as you have enough memory you should be Ok. Everything is running in userspace. This runs a bit counterintuitive to me, and also is contrary to one of the positive points made about block storage--decoupling. But it's a valid use case.

## From Folsom to Grizzy: A DevOps Upgrade Pattern

This was the last session of the day, and I was quite tired, so apologies. Suffice it to say: _Even imagining upgrading OpenStack is difficult and requires a lot of bullet points_.

## Misc Notes from the Day
<p></p>
- There's free sunshine and *stout* outside on the balcony
- [Probability of data loss in Swift](http://www.buildcloudstorage.com/2012/08/is-openstack-swift-reliable-enough-for.html) (this is not to say it's high, rather an analysis)
- Infiniband doesn't have a lot of traction in OpenStack AFAIK, need to look into this
- San Diego has basically the same weather all the time
- Japan's summers are very hot and humid
- !https://fbcdn-sphotos-e-a.akamaihd.net/hphotos-ak-ash4/430171_10150245244739982_1571995757_n.jpg!


