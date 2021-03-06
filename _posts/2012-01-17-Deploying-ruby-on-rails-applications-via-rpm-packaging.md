---
layout: post
title: Deploying Ruby-on-Rails applications using RPM packaging 
categories:
- ruby

---

# {{ page.title }} 

It's been a long time between posts but the time has come!

In this post I hope to take a good look at one way to deploy a working ruby on rails (RoR) application by packaging it in an RPM.

In this example all of the gems the application requires are downloaded and built/compiled at the same time the RPM is, and thus the RPM contains all the required gems (100+ in this example). The best way to deploy an application, in my opinion, would be to standardize on a set of gems that is available at the OS level--so the RPM would not contain _any_ gems, rather would require the general OS level gems.

Unfortunately, for many reasons, which I won't get into, that is just not possible for me at this time. Maybe in the future when all gems can easily be built into RPMs, and also when internal developers can agree on a set of gems. Someday...

## Environment

We're deploying to a specific RHEL6 server environment.

### Ruby version

We'll be deploying the RoR application to [Redhat Enterprise 6](http://distrowatch.com/table.php?distribution=redhat) (RHEL6) virtual machine which has, *and likely always will have*, @ruby 1.8.7@ (with backported security patches of course!).

<pre>
<code>[root@RoR-TEST ~]# ruby -v
ruby 1.8.7 (2010-06-23 patchlevel 299) [x86_64-linux]
[root@RoR-TEST ~]# cat /etc/redhat-release 
Red Hat Enterprise Linux Server release 6.1 (Santiago)
</code>
</pre>

This will likely be a problem in the future, as it seems that Rails 3.2 will be the last version that supports ruby 1.8.X (where X seems to be 7+ as 1.8.6 is specifially not supported). At some point the dev team may want to go to a Rails version that will not run on Ruby 1.8.7.

### Apache and passenger

We'll also be deploying the RoR app using apache and passenger. 

## Requirements

A few things are required to build and deploy an RPM.

# The application code in some kind of version control system and hopefully that VCS supports tagging...svn, mercurial, and git all support tags.
# A build server that is the same as OS and arch as the production server being deployed to. In this case, RHEL6 and X86_64.
** A spec file for the application.
** This build server needs `bundle` and `gem` available in the binary PATH because currently the example spec file needs it to be there.
** A working rpmbuild environment, configured as appropriate.
# A test server to test the RPM deployment, ie. a place to actually install the RPM into.

## The spec file

Building a RPM requires, among other things, a spec file. This file is the heart of a RoR RPM deployment.

I have put an example spec file up on [github](https://github.com/ccollicutt/Ruby-on-Rails-Example-RPM-Deployment-spec-file) to peruse and abuse. Again, it's not going to work out of the box, but it's a good example, or will be at some point. :)

The build portion of the spec file is what is interesting in terms of deploying a RoR app with RPM.

Prior to the build section the code has been pulled out of a git repository into a local build directory by the rpmbuild process.

In the build section, which I'm cutting and pasting examples out of, we are going to cd into that checked out repository and use bundle to *compile and install* all the gems into `./vendor/bundle`.

<pre>
<code>%build
pushd %{name}

# Install all required gems into ./vendor/bundle using the handy bundle commmand
bundle install --deployment
</code>
</pre>

Once that has completed, which could be quite a long process depending on the number and complexity of the gems required, we remove the assets and recompile them.

<pre>
<code># Compile assets, this only has to be done once AFAIK, so in the RPM is fine
rm -rf ./public/assets/*
bundle exec rake assets:precompile

</code>
</pre>

Then we need to also build bundler into the RPM as well, which requires a smidge of trickery:

<pre>
<code># For some reason bundler doesn't install itself, this is probably right,
# but I guess it expects bundler to be on the server being deployed to
# already. But the rails-helloworld app crashes on passenger looking for
# bundler, so it would seem to me to be required. So, I used gem to install
# bundler after bundle deployment. :) And the app then works under passenger.

PWD=`pwd`
cat > gemrc <<EOGEMRC
gemhome: $PWD/vendor/bundle/ruby/1.8
gempath:
- $PWD/vendor/bundle/ruby/1.8
EOGEMRC
        #gem --source %{gem_source} --config-file ./gemrc install bundler
        gem --config-file ./gemrc install bundler
# Don't need the gemrc any more...
rm ./gemrc

</code>
</pre>

Finally, it seems that some of the gems have a funny location for ruby set, which we need to change because the rpmbuild process will mark that as a requirement. This issue may be fixed now.

<pre>
<code># Some of the files in here have /usr/local/bin/ruby set as the bang
# but that won't work, and makes the rpmbuild process add /usr/local/bin/ruby
# to the dependencies. So I'm changing that here. Either way it prob won't
# work. But at least this rids us of the dependencie that we can never meet.
for f in `grep -ril "\/usr\/local\/bin\/ruby" ./vendor`; do
         sed -i "s|/usr/local/bin/ruby|/usr/bin/ruby|g" $f
         head -1 $f
done

popd
</code>
</pre>

Basically, three major things happen in the build section:

# Use the handy bundler application to install all the required gems
# Also install bundler itself
# Work around other issues as found

Once that is done, we have a nice spec file that can be built and then installed!

## rpmbuild

Now we build our RPM. In this example I'm building a RoR application called `special_collections`. `rhel6b` is my RHEL6 build server/environment. 

<pre>
<code>[curtis@rhel6b SPECS]$ rpmbuild -ba special_collections.spec 
Executing(%prep): /bin/sh -e /var/tmp/rpm-tmp.J1hbLc
+ umask 022
+ cd /home/curtis/rpmbuild/BUILD
+ rm -rf ./special_collections
+ git clone https://code.example.com/git/special_collections
Initialized empty Git repository in /home/curtis/rpmbuild/BUILD/special_collections/.git/
SNIP!
Checking for unpackaged file(s): /usr/lib/rpm/check-files /home/curtis/rpmbuild/BUILDROOT/special_collections-0.1.4-1.el6.ualib.x86_64
Wrote: /home/curtis/rpmbuild/SRPMS/special_collections-0.1.4-1.el6.ualib.src.rpm
Wrote: /home/curtis/rpmbuild/RPMS/x86_64/special_collections-0.1.4-1.el6.ualib.x86_64.rpm
Executing(%clean): /bin/sh -e /var/tmp/rpm-tmp.VOkPMU
+ umask 022
+ cd /home/curtis/rpmbuild/BUILD
+ rm -rf /home/curtis/rpmbuild/BUILDROOT/special_collections-0.1.4-1.el6.ualib.x86_64
+ exit 0
</code>
</pre>

*NOTES:* 

- The above rpmbuild could take a long time depending on the number of gems that the application requires. It's important to rembember that in this process all the gems are being downloaded from [rubygems.org](http://rubygems.org) and then also _compiled_ on the build server, each and every time the rpm is built. So it's slow. There are some things I'm looking at doing to reduce the time it takes to build the RPM, but that's where it is right now. Maybe someone will read this blog and give me some comments on what I can be doing better!
- The resulting RPM is quite large...in this case about 80MB _compressed_. This is because it has 100+ gems in it.

## Installing the RPM on a brand new server

I have a brand new server all ready for this ruby application to be deployed. It's a minimal install.

<pre>
<code>[root@RoR-TEST ~]# rpm -qa | grep -i "apache\|ruby\|passenger"
[root@RoR-TEST ~]# 
# Nothing! No ruby, passenger, or apache currently installed.
[root@RoR-TEST ~]# rpm -qa | wc -l
293
# And only 293 RPMs!
</code>
</pre>

Normally I install a RPM from a custom yum repository, but in this example I will use @yum localinstall@ so I copy the RPM from the build server to the new server.

Note that I have several 3rd party repositories configured on this server, including epel, rpmforge, and the passenger repository. Obviously one has to trust a 3rd party repository to use it. Configuring yum priorities might be a good idea as well to try to avoid unwanted collisions.

So, to install:

<pre>
<code>[root@RoR-TEST tmp]# yum localinstall special_collections-0.1.4-1.el6.ualib.x86_64.rpm 
SNIP!
 rubygem-passenger-native-libs  x86_64  1:3.0.11-1.el6_1.8.7.352   passenger                                       29 k
 rubygem-rack                   noarch  1:1.1.0-2.el6              epel                                           446 k
 rubygem-rake                   noarch  0.8.7-2.1.el6              optional                                       403 k
 rubygems                       noarch  1.3.7-1.el6                optional                                       206 k
 sgml-common                    noarch  0.6.3-32.el6               base                                            43 k

Transaction Summary
========================================================================================================================
Install      73 Package(s)

Total size: 234 M
Total download size: 74 M
Installed size: 413 M
Is this ok [y/N]: y
SNIP!
  rubygem-daemon_controller.noarch 0:0.2.6-1.el6                   rubygem-fastthread.x86_64 0:1.0.7-2.el6             
  rubygem-passenger.x86_64 1:3.0.11-1.el6                          rubygem-passenger-native.x86_64 1:3.0.11-1.el6      
  rubygem-passenger-native-libs.x86_64 1:3.0.11-1.el6_1.8.7.352    rubygem-rack.noarch 1:1.1.0-2.el6                   
  rubygem-rake.noarch 0:0.8.7-2.1.el6                              rubygems.noarch 0:1.3.7-1.el6                       
  sgml-common.noarch 0:0.6.3-32.el6                               

Complete!
</code>
</pre>
<br />

## Configure the application

Currently the RPM will create a directory in `/etc/` that contains the `database.yml` file for the rails app:

<pre>
<code>[root@RoR-TEST special_collections]# pwd
/etc/railsapps/special_collections
[root@RoR-TEST special_collections]# ls
database.yml
</code>
</pre>


Edit that to set the proper database information.

## Configure apache

Now that apache has been installed because it is required by the custom RPM it needs to be configured.

First let's make sure it'll start on a reboot. Don't want to have to login on the weekend three months from now after a spontaneous reboot now do we? :)

<pre>
<code>[root@RoR-TEST yum.repos.d]# chkconfig httpd on
</code>
</pre>

Now to setup the apache rails environment for this particular application. Note that in this case, we're doing one RoR app per virtual host. It's just easier for me because there are some variables that need to be set in the virtual host config file.

I also always configure a `/etc/httpd/conf.d/vhost.d` directory for virtual host files, and tell httpd to check there for `*.conf` files.

<pre>
<code>[root@RoR-TEST vhost.d]# grep vhost.d /etc/httpd/conf/httpd.conf 
Include conf.d/vhost.d/*.conf
</code>
</pre>

The vhost config file looks like this:

<pre>
<code>[root@RoR-TEST vhost.d]# cat specialcollections.example.com.conf 
<VirtualHost *:80>
   ServerName specialcollections.example.com
   DocumentRoot /usr/share/railsapps/special_collections/public

   # Because of the way we're deploying rails apps, ie. by using bundler during the rpm
   # build process to install all the required gems into $RAILSAPP/$NAME/vendor/bundle/ruby/1.8
   # this has to be set here. Otherwise the app will not have the required gems to run.
   SetEnv GEM_HOME /usr/share/railsapps/special_collections/vendor/bundle/ruby/1.8/
   <Directory /usr/share/railsapps/special_collections/public>
        Options -MultiViews
    </Directory>
</VirtualHost>
</code>
</pre>

Startup apache:

<pre>
<code>[root@RoR-TEST vhost.d]# service httpd configtest
[root@RoR-TEST vhost.d]# service httpd start
</code>
</pre>

Done with apache.

## Rake

Now to configure the initial database.

First, the paths need to be setup. I create a file called `special_collectionsrc` that has path information setup. Note that this rc file is someting I created specifically for this application because each rails app will have it's own paths _and_ gems. Then, when wanting to use rake with the specific application that file is sourced to ensure the correct rake and other gems are used.

<pre>
<code>[root@RoR-TEST ~]# which rake
/usr/bin/rake
# oops not the right one!
[root@RoR-TEST ~]# which bundle
/usr/bin/which: no bundle in (/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin)
# oops isn't on the path!
[root@RoR-TEST ~]# cat special_collectionsrc 
#!/bin/bash
export GEM_HOME=/usr/share/railsapps/special_collections/vendor/bundle/ruby/1.8
PATH=/usr/share/railsapps/special_collections/vendor/bundle/ruby/1.8/bin:$PATH
export RAILS_ENV=production
</code>
</pre>

Once that file is sourced, we should be able to find rake on the path:

<pre>
<code>[root@RoR-TEST ~]# source special_collectionsrc 
[root@RoR-TEST ~]# which rake
/usr/share/railsapps/special_collections/vendor/bundle/ruby/1.8/bin/rake
[root@RoR-TEST ~]# which bundle
/usr/share/railsapps/special_collections/vendor/bundle/ruby/1.8/bin/bundle
</code>
</pre>

cd to `/usr/share/railsapps/special_collections/` and load the db:

<pre>
<code>[root@RoR-TEST special_collections]# rake db:load
/usr/share/railsapps/special_collections/vendor/bundle/ruby/1.8/gems/curb-0.7.16/lib/curb_core.so: warning: already initialized constant CURL_SSLVERSION_DEFAULT
-- create_table("collections", {:force=>true})
   -> 0.4194s
-- create_table("gallery_images", {:force=>true})
   -> 0.0040s
-- initialize_schema_migrations_table()
   -> 0.0077s
-- assume_migrated_upto_version(20111104163654, ["db/migrate"])
   -> 0.0048s
</code>
</pre>

Whenever working with this particular RoR app the rc file should be sourced.

Done raking.

## That's...it

At this point the rails app should be available at the virtual host URL that was configured in the vhost. :)

While it's a long process to get that intial spec file and rpmbuild working, once it's done the application can be deployed in a few minutes, and now the developers can simply worry about commiting and tagging code, and let the sysadmin deal with deploying the actual application in a replicable manner. Of course there will be some back and forth, new gems might not compile, etc, but the general structure is in place. Further, the deployment is quite automatable--a new tag could mean a new RPM build and deployment to test.

