---
layout: post
title: Rabbitmq Server with SSL/TLS 
categories:
header_image: /img/secure_rabbit.jpg
---

# {{ page.title }}

For some reason I am a glutton for punishment as I try to "TLS enable all the things" which doesn't always work out. Note that the [rabbitmq documentation](https://www.rabbitmq.com/ssl.html) for SSL/TLS is pretty good; I'm not showing here much more than you can get from that, but I thought I'd post it anyway. :)

Anyways, one of the more interesting things I've enabled TLS on lately is Rabbitmq. What's more this is in production right now and is working fine. There is some debate as to whether or not it's a good idea to do TLS with Rabbitmq, especially if it's an internal only queue, but I think it's always best to encrypt when we can. I suppose there is the possibility of performance issues, but I don't mind throwing hardware at it. I should also note that I'm just doing "over the wire" encryption. The certificates aren't being used for authentication. (Future work.)

This is an example configuration for a three node Rabbitmq cluster:

<pre>
<code>[
 {ssl, [{versions, ['tlsv1.2', 'tlsv1.1']}]},
 {rabbit, [
           {cluster_nodes, ['rabbit`node2','rabbit`node5']},
           {tcp_listeners, []},
           {ssl_listeners, [{"192.168.0.12",5671}]},
           {ssl_options, [{cacertfile,"/etc/ssl/certs/example.com-intermediate.crt"},
                          {certfile,  "/etc/ssl/private/example.com.crt"},
                          {keyfile,   "/etc/ssl/private/example.com.key"},
                          {versions, ['tlsv1.2', 'tlsv1.1']}
                         ]}
          ]}
].
</code>
</pre>

This is the contents of the rabbitmq-env.conf file, not that it should be necessary in most cases. Sometimes I name the node something different than the hostname, or perhaps internal communication only happens on a specific VLAN so it has to listen on a specific interface.

<pre>
<code>export RABBITMQ_NODENAME=rabbit@node2
export RABBITMQ_NODE_IP_ADDRESS=192.168.0.12
</code>
</pre>

Then rabbitmq will be listening on port 5671.

<pre>
<code>ubuntu@node2:/etc/rabbitmq$ sudo lsof -i -nP | grep LISTEN | grep ":5671"
beam.smp  18935  rabbitmq   20u  IPv4  165304426      0t0  TCP 192.169.0.12:5671 (LISTEN)
</code>
</pre>

Now any of your applications that are using the rabbitmq queue can connect via TLS on port 5671.

There's a lot more work to be done here, but it's a start!
