---
layout: post
title: 36 hot swappable hard-drive bay Supermicro server specs
categories:
---

# {{ page.title }}

![](https://github.com/ccollicutt/ccollicutt.github.com/raw/master/img/supermicro_stack_small.jpg)

The above is the front of four Supermicro 36-bay chassis servers racked but not completely filled with drives.

![](https://github.com/ccollicutt/ccollicutt.github.com/raw/master/img/supermicro_back_small.jpg)

This is the back of the server. As you can see, there are 12 hot swappable slots in the back. The motherboard has room for seven low profile boards of varying PCIe speeds.

 Sorry for the poor picture quality--I don't get paid enough to have a phone with a good camera. ;)

This is output from the bottom server that has four drives lit up:

<pre>
<code># ls /dev/sd?
/dev/sda  /dev/sdb  /dev/sdc  /dev/sdd  /dev/sde  /dev/sdf
</code>
</pre>

so with the two internal hard-drives that makes six, which is what we see above. If the server was filled up with drives, there would be 38.

## First things First: The Specs

Why not get this out of the way? That's what you're here for, isn't it? :) The below would be for one server:

| *Item*  | *Part or Part #* | *Quantity* |
| 4U 36 bay Chassis | SM [SC847E16-R1400LPB](http://www.supermicro.com/products/chassis/4u/847/sc847e16-r1400lp.cfm) | 1 |
| iPass cable | SM CBL-0281L | 1 |
| iPass cable | SM 0108L-02 | 1 |
| Internal HD Brackets | SM MCP-220-84701-0N | 2 |
| Motherboard | SM X8DT6-F | 1 |
| CPU | Xeon E5645 (6 cores) | 2 |
| Heat Sink/CPU Fan | Dynatron G666 | 2 |
| Kingston 24GB Pack  | SM KVR1066D3Q8R7SK3/24G | 2 |
| SATA power splitter |	SM CBL-0082L | 1 |

<br>
The total cost for the above is around *$4000*.

What you get is one 4U, dual CPU server (24 cores counting hyperthreading), with 48GB of RAM, the MB supports up to 196GB I believe, and room for 36 hot swappable drives, PLUS two internal OS drives that are actually bracketed deep inside the chassis, ie. difficult to get out. 

%{color:blue}Note:% I've done my best to make sure the above is correct, and I do have exactly that hardware, but don't base your business on those specs without doing some testing. Order one server and see how it goes before ordering a dozen+. 

%{color:blue}Note:% You would need to find a local vendor to put the pieces together, or you could do it yourself--but I don't recommend that. I'd put 5% of the total cost aside for professional assembly. We were lucky enough that our vendor was willing to put them together for free--at least on the first order. :)

![](https://github.com/ccollicutt/ccollicutt.github.com/raw/master/img/supermicro_internal_hard_drive_small.jpg)

(Terrible image...but you can kind of see the two internal hard-drives.)

So with those two internal slots, plus 24 bays in the front, and 12 in the back is a total of 38 3.5" slots, 36 of them hot swappable. Or, put another way, if using 3TB SATA drives...114TB of raw storage. (Also I think you can fit four 2.5" drives internally instead of two 3.5" drives. But don't quote me on that.)

## Bulk storage costs

The amount of data an organization has to store never seems to go down, only up up UP!--probably at least 50% per year. High-end, "enterprise storage", can cost between $50K-$100K per terabyte, where K is thousands. 

I'm not kidding. It's _really_ that much, and there are few organizations that can afford to have their storage increase by 50% per year at that cost. Everywhere I've worked has started out with a large Enterprise SAN and quickly realized they can't afford it.

Of course, there are reasons why enterprise storage costs so much. It's not easy to design and build a large, production, high-performance SAN, and in most cases trying to replicate one with commodity hardware is not a good idea, and could cost people their jobs. Not a nice thought, but it's true.

But in some specialized cases, such as bulk data storage, _not_ using an Enterprise SAN might be an option. However, unless you work at Facebook, or another company that somehow has access to hardware based on Open Compute's [Open Vault](http://opencompute.org/project_category/storage-technology/), what do you do for bulk storage hardware?

One possibility is the above 36-bay Supermicro chassis based server. I know there are people out there using them. _Lots_ of them.

## Now you can afford test infrastructure

One note I feel important to make is that when you purchase cost effective hardware there is often money on the table to buy more hardware for test. I always like to have test hardware running so that I don't have to worry about "trying something out" in production. When running production systems I never want to be afraid to make a change, and the way to do that is to have test infrastructure to experiment and...test with.

If your enterprise hardware is so expensive that test infrastructure is completely out of the question, then I wonder how easy it will be to try things out; to make changes without fear of brining down the production system?

To me there is massive value in test hardware.

Also, test hardware can be moved from test to production in case of failure.

## Support

Obviously if you spec your own hardware, versus say buying from a tier one vendor, support means that if it breaks you pull a new part off the shelf or out of test and replace it in production yourself.

Further, tier one vendors will usually certify their hardware for particular operating systems and applications. You won't have that if you spec your own. But you certainly can start small (and inexpensively) and test out your specs/OS/applications, then go bigger when you know it all works well together.

Personally, I don't like spending 3 hours on the phone with the tier one support, going through generic call scripts with someone who doesn't care about my problem in the least, and then having the local support tech either be a "cell phone with hands" or cancel four times in a row before simply replacing the whole motherboard because they have no idea what is actually wrong. I suppose that sounds kind of negative, but those are real anecdotes. :) PS. We have lots of tier one vendor servers...

## Bonus

The Java IPMI remote console gui works great on Linux! Some tier one vendors don't...and they make you pay extra to actually use the remote console. (*cough* HP *cough*)

That said...don't look at how they store passwords in the ssh interface. Just don't.

## Conclusion

I feel that in specialized applications systems like this supermicro server can be very effective. Especially if it leaves money on the table for test infrastructure. I'd rather have twelve (nine production, three in test) of these boxes than two four-socket tier one servers. 

I recommend using 5% of the budget for parts to go on the shelf, and 5%-25% for test.

If you have any questions/concerns, please comment.
