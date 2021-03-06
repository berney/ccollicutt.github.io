---
layout: post
title: Ansible Custom Facts
header_image: /img/ap150127.jpg
categories:
---

_([nasa image](http://apod.nasa.gov/apod/ap150127.html)_)

# {{ page.title }}

I like Ansible. It's not perfect but nothing is. Recently I was playing around with the [tinc vpn](https://github.com/ccollicutt/ansible-tinc) system and wanted a way to set a custom fact per virtual machine based on the vms public ip address. I figured the best way to do that would be to setup a custom fact. It turns out there isn't that much documentation on just how to do that, or I simply can't find it. So I'm going to describe what I did in order to setup and use a custom fact.

## A bash script

I just wanted a simple script to parse the ip address of the server and return a private ip based on the last octet of the address.

This is the script I ended up using.

~~~bash
#!/bin/bash

IP_INDEX=`hostname -i | cut -f 4 -d "."`

cat <<EOF
{
    "vpn_ip" : "10.0.0.$IP_INDEX"
}
EOF
~~~

Using ansible I put that file into the /etc/ansible/facts.d directory. Note it has to be executable, return json, and end with the .fact extension.

~~~bash
ubuntu@trusty2:~$ cd /etc/ansible/facts.d/
ubuntu@trusty2:/etc/ansible/facts.d$ ./tinc_facts.fact
{
    "vpn_ip" : "10.0.0.22"
}
~~~

Pretty simple so far.

## Loading the custom fact

So, the *first time* that file is loaded onto the server Ansible setup won't have it yet (unless it was loaded up in a previous role)  so if you do load the facts file in the same role you'll have to use setup in the task list to load the custom fact.

Here's a snippet of my playbook. It's creating the facts.d directory, copying over the script, and finally re-running setup with the ansible_local filter.

I should be registering a variable and only reload the ansible_local if the facts.d scripts have changed on this particular run.

~~~bash
- name: ensure custom facts directory exists
  file: >
    path=/etc/ansible/facts.d
    recurse=yes
    state=directory

- name: install custom fact module for IP address
  template: >
    src=tinc_facts.sh.j2
    dest=/etc/ansible/facts.d/tinc_facts.fact
    mode=0755

- name: reload ansible_local
  setup: filter=ansible_local
~~~

Otherwise, on the next Ansible run it'll be available.

## Using the fact

It seems as though the fact gets loaded into the ansible_local namespace under the name of the name of the fact script.

~~~bash
$ ansible -m setup trusty2 | grep -A 4 ansible_local
        "ansible_local": {
            "tinc_facts": {
                "vpn_ip": "10.0.0.22"
            }
        },
~~~

I use the fact like this in my playbook.

~~~bash
{% raw %}- name: ensure subnet ip address is properly set in tinc host file
  lineinfile: >
    dest=/etc/tinc/{{ vpn_name }}/hosts/{{ ansible_hostname }}
    line="Subnet = {{ ansible_local.tinc_facts.vpn_ip }}/32"
    create=yes
  notify:
    - restart tinc
{% endraw %}
~~~

And there we go, relatively simple custom facts for Ansible. Now we can do all sorts of silly things like make new ips based on old ips. haha.
