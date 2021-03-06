---
layout: post
title: Notes on MySQL, MariaDB, and Galera
categories:
header_image: /img/bigdata.jpg
header_permalink: https://www.flickr.com/photos/kamiphuc/11396381123/in/photolist-in4r4H-eAK6u-askS2i-gbvjJD-dHo3mh-avnpd5-imfDLH-fh7UPq-6UZFV4-7cDQve-8H9Sw-8yyKdK-aZhu44-9w3DwZ-bUY1mb-ddn7sf-7n3F19-c8grpb-qStVmK-e5vYVa-aid5CB-pFUvT2-ddn4Pj-vxGcGu-6AJLvS-eQczcB-ddmYSc-eQoYFN-dZ4QZP-ddn3jP-9E8gsW-nHxGvf-7mYLL4-9Ei5gW-ekYLQJ-5oRn8Z-nMndi9-nPjzct-nMf1qp-rJZkZ1-6V4LRL-9enWWG-Ghm6-puR46m-kwvN7x-kwxuNS-in4qSv-kwxuE5-mt36Ls-kwvKtX
---

# {{ page.title }}

This is just a set of random notes on MySQL, MariaDB, Galera, other database related thoughts, and googling results for test databases and performance testing MySQL/MariaDB as I work towards getting a better understanding of my MariaDB galera cluster.

## MariaDB Cluster

I've got a MariaDB cluster made up (well, as of right this moment) 3 nodes on one network and a garbd on another. This is not the right way to do this, but again, this is just a set of notes. Eventually I'll have 3 networks (in test) that are meant to represent 3 data centers. Two networks will have 3 nodes on each (for a total of 6) and another network a single garbd server. What I'm aiming for is for any of the DCs to go down and to still have a working cluster in the remaining DC. The nodes are all virtual machines.

I'm a bit behind on my MariaDB version, so I'm still back on 5.5, but that is what is in production right now so that is what I need to test with.

<pre>
<code>root@mariadb-1:/home/ubuntu# dpkg --list | grep mariadb
ii  libmariadbclient18                5.5.45+maria-1~trusty            amd64        MariaDB database client library
ii  mariadb-client                    5.5.45+maria-1~trusty            all          MariaDB database client (metapackage depending on the latest version)
ii  mariadb-client-5.5                5.5.45+maria-1~trusty            amd64        MariaDB database client binaries
ii  mariadb-client-core-5.5           5.5.45+maria-1~trusty            amd64        MariaDB database core client binaries
ii  mariadb-common                    5.5.45+maria-1~trusty            all          MariaDB database common files (e.g. /etc/mysql/conf.d/mariadb.cnf)
ii  mariadb-galera-server-5.5         5.5.45+maria-1~trusty            amd64        MariaDB database server with Galera cluster binaries
</code>
</pre>

## OpenStack Ansible Galera playbooks

I (internally) forked the OpenStack Ansible Galera playbooks some time ago. They are a good way to get a MariaDB Galera cluster up and running quickly. The roles can easily be found on the [github site](https://github.com/openstack/openstack-ansible/tree/master/playbooks/roles).

## Galera-arbitrator (arbiter?)

Galera-arbitrator (garb or garbd) is a useful service that can help a Galera cluster with maintaining quorum, but doesn't take up as many resources to run it as it would a full-fledged database server. Usually people who only have two good database servers use garbd on a lower-end server to help with quorum because you shouldn't have a cluster of two nodes or you'll end up in split-brain, and split-brain is as bad as it sounds. So if you have MariaDB + Galera on two good servers and garbd on a third (less good) server, then you should be able to avoid split-brain.

In my case I have two datacenters with multiple galera nodes in a large cluster, and I want a garbd running in a third datacenter so that if I lose an entire DC, or the interconnect between them, I don't end up in split-brain at the datacenter level.

<pre>
<code>ubuntu@garb-1:~$ dpkg --list | grep galera
ii  galera-arbitrator-3               25.3.9-trusty                    amd64        Galera arbitrator daemon
</code>
</pre>

This is what my /etc/default/garb looks like. Again there are four nodes, which isn't quite correct, but I'm in testing mode. :)

<pre>
<code>ubuntu@garb-1:/etc/default$ cat garb 
# Copyright (C) 2012 Codership Oy
# This config file is to be sourced by garb service script.

# A space-separated list of node addresses (address[:port]) in the cluster
GALERA_NODES="192.168.77.6:4567 192.168.44.34:4567 192.168.44.33:4567 192.168.44.32:4567"

# Galera cluster name, should be the same as on the rest of the nodes.
GALERA_GROUP="rpc_galera_cluster"

# Optional Galera internal options string (e.g. SSL settings)
# see http://www.codership.com/wiki/doku.php?id=galera_parameters
# GALERA_OPTIONS=""

# Log file for garbd. Optional, by default logs to syslog
# LOG_FILE=""
</code>
</pre>

This is what the wsrep_incoming_addresses looks like on one of the mariadb nodes.

<pre>
<code>MariaDB [(none)]> show status like 'wsrep_incoming_addresses';
+--------------------------+-----------------------------------------------------------+
| Variable_name            | Value                                                     |
+--------------------------+-----------------------------------------------------------+
| wsrep_incoming_addresses | 192.168.44.33:3306,,192.168.44.34:3306,192.168.44.32:3306 |
+--------------------------+-----------------------------------------------------------+
1 row in set (0.00 sec)
</code>
</pre>

Lots of interesting information. :)

## Test (fake) databases

One that I found is the [employees](https://launchpad.net/test-db/employees-db-1/1.0.6/+download/employees_db-full-1.0.6.tar.bz2) database. I believe I stumbled upon the existence of the employees test database through [this post](https://www.digitalocean.com/community/tutorials/how-to-measure-mysql-query-performance-with-mysqlslap).

Once you download that and unbzip it, you'll have these files.

<pre>
<code>ubuntu@mariadb-1:~/employees_db$ ls
Changelog                   employees_partitioned.sql  load_dept_emp.dump      load_salaries.dump  README
employees_partitioned2.sql  employees.sql              load_dept_manager.dump  load_titles.dump    test_employees_md5.sql
employees_partitioned3.sql  load_departments.dump      load_employees.dump     objects.sql         test_employees_sha.sql
</code>
</pre>

Then you can simply import the database using:

<pre>
<code>$ mysql < employees.sql
</code>
</pre>

For example, the salaries table has quite a few entries.

<pre>
<code>MariaDB [employees]> select count(*) from salaries;
+----------+
| count(*) |
+----------+
|  2844047 |
+----------+
1 row in set (0.70 sec)
</code>
</pre>


## MySQL Procedures

While I was looking for test databases, I stumbled on [this stackoverflow post](https://stackoverflow.com/questions/3766282/fill-database-tables-with-a-large-amount-of-test-data) that had an example prepared statement in it. I figured why not give it a try, I'd never used a prepared statement in MySQL before. Another technology to look into...

<pre>
<code>root@mariadb-1:/home/ubuntu# cat fake_data.sql 
CREATE TABLE your_table (id int NOT NULL PRIMARY KEY AUTO_INCREMENT, val int);
DELIMITER $$
CREATE PROCEDURE prepare_data()
BEGIN
  DECLARE i INT DEFAULT 100;

  WHILE i < 100000 DO
    INSERT INTO your_table (val) VALUES (i);
    SET i = i + 1;
  END WHILE;
END$$
DELIMITER ;
-- CALL prepare_data()
</code>
</pre>

All it's going to do is create a table called "your_table" and load ~100000 entries into it.

I ran it a few times to try it out.

<pre>
<code>MariaDB [fake_data]> select count(*) from your_table;
+----------+
| count(*) |
+----------+
|   299700 |
+----------+
1 row in set (0.16 sec)
</code>
</pre>

Here's how to list the procedures.

<pre>
<code>MariaDB [fake_data]> SHOW PROCEDURE STATUS;
+-----------+--------------+-----------+---------+---------------------+---------------------+---------------+---------+----------------------+----------------------+--------------------+
| Db        | Name         | Type      | Definer | Modified            | Created             | Security_type | Comment | character_set_client | collation_connection | Database Collation |
+-----------+--------------+-----------+---------+---------------------+---------------------+---------------+---------+----------------------+----------------------+--------------------+
| fake_data | prepare_data | PROCEDURE | root@%  | 2015-09-25 22:27:16 | 2015-09-25 22:27:16 | DEFINER       |         | utf8                 | utf8_general_ci      | utf8_unicode_ci    |
+-----------+--------------+-----------+---------+---------------------+---------------------+---------------+---------+----------------------+----------------------+--------------------+
1 row in set (0.02 sec)
</code>
</pre>


## Size of databases

Here's one way to get the size of the databases in your MySQL/MariaDB cluster. This was borrowed from [this stackoverflow post](https://stackoverflow.com/questions/1733507/how-to-get-size-of-mysql-database). (I guess I use stackoverflow questions/answers more than I thought.)

<pre>
<code>MariaDB [(none)]> SELECT table_schema "table name", sum( data_length + index_length ) / 1024 / 1024 "Data Base Size in MB" 
    -> FROM information_schema.TABLES GROUP BY table_schema ; 
+--------------------+----------------------+
| table name         | Data Base Size in MB |
+--------------------+----------------------+
| employees          |         197.43750000 |
| fake_data          |           8.51562500 |
| information_schema |           0.15625000 |
| mysql              |           0.62678719 |
| performance_schema |           0.00000000 |
| sysbench           |        4752.00000000 |
+--------------------+----------------------+
6 rows in set (0.14 sec)
</code>
</pre>

## sysbench

I put up a quick Ansible playbook that installs the lastest sysbench [here](https://gist.github.com/ccollicutt/2a96ca4f03a7f18b9da9). Currently that is version 0.5. Apparently 0.5 adds the ability to use lua scripts, and in fact comes with some example scripts which I use below.

Following [this post](http://blog.secaserver.com/2014/07/sysbench-0-5-ubuntu-14-04-trusty-percona-server-xtradb-cluster/) I setup and ran tests using the below commands (where all the right databases and users and permissions and such were put into place).

<pre>
<code>ubuntu@mysql-client-1:/usr/local/bin$ cat sysbench-prepare-test.sh 
#!/bin/bash

sysbench \
--db-driver=mysql \
--mysql-table-engine=innodb \
--oltp-table-size=20000000 \
--mysql-host=192.168.44.34 \
--mysql-db=sysbench \
--mysql-port=3306 \
--mysql-user=sysbench \
--mysql-password=syb3nch \
--test=/usr/local/src/sysbench/sysbench/tests/db/oltp.lua \
prepare
</code>
</pre>

20000000 records is probably way to many for the size of servers I'm using now (which is about 4 gigs of memory per MariaDB node).

This is the test run script:

<pre>
<code>ubuntu@mysql-client-1:/usr/local/bin$ cat sysbench-run-test.sh 
#!/bin/bash

sysbench \
--db-driver=mysql \
--num-threads=8 \
--max-requests=50000 \
--oltp-table-size=20000000 \
--oltp-test-mode=complex \
--test=/usr/local/src/sysbench/sysbench/tests/db/oltp.lua \
--mysql-host=192.168.44.34 \
--mysql-db=sysbench \
--mysql-port=3306 \
--mysql-user=sysbench \
--mysql-password=syb3nch \
run
</code>
</pre>

And the results of running that test:

<pre>
<code>ubuntu@mysql-client-1:/usr/local/bin$ ./sysbench-run-test.sh 
sysbench 0.5:  multi-threaded system evaluation benchmark

Running the test with following options:
Number of threads: 8
Random number generator seed is 0 and will be ignored


Threads started!

OLTP test statistics:
    queries performed:
        read:                            700000
        write:                           200000
        other:                           100000
        total:                           1000000
    transactions:                        50000  (309.68 per sec.)
    read/write requests:                 900000 (5574.19 per sec.)
    other operations:                    100000 (619.35 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          161.4584s
    total number of events:              50000
    total time taken by event execution: 1291.3610s
    response time:
         min:                                  7.33ms
         avg:                                 25.83ms
         max:                                172.26ms
         approx.  95 percentile:              43.51ms

Threads fairness:
    events (avg/stddev):           6250.0000/118.29
    execution time (avg/stddev):   161.4201/0.01

</code>
</pre>

When I dropped the number of entries to 50000, these are my results.

<pre>
<code>ubuntu@mysql-client-1:/usr/local/bin$ ./sysbench-run-test.sh 
sysbench 0.5:  multi-threaded system evaluation benchmark

Running the test with following options:
Number of threads: 8
Random number generator seed is 0 and will be ignored


Threads started!

^[9OLTP test statistics:
    queries performed:
        read:                            700000
        write:                           200000
        other:                           100000
        total:                           1000000
    transactions:                        50000  (270.09 per sec.)
    read/write requests:                 900000 (4861.63 per sec.)
    other operations:                    100000 (540.18 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          185.1233s
    total number of events:              50000
    total time taken by event execution: 1480.6339s
    response time:
         min:                                  7.53ms
         avg:                                 29.61ms
         max:                                269.02ms
         approx.  95 percentile:              50.34ms

Threads fairness:
    events (avg/stddev):           6250.0000/158.26
    execution time (avg/stddev):   185.0792/0.01
</code>
</pre>

For some kind of comparison, good or bad, here's the same test run on a single instance of the default mysql server you get when you install it on Ubuntu trusty. Same instance type as the above tests were run on.

<pre>
<code>ubuntu@mysql-client-1:/usr/local/bin$ ./sysbench-run-test.sh 
sysbench 0.5:  multi-threaded system evaluation benchmark

Running the test with following options:
Number of threads: 8
Random number generator seed is 0 and will be ignored


Threads started!

OLTP test statistics:
    queries performed:
        read:                            700000
        write:                           200000
        other:                           100000
        total:                           1000000
    transactions:                        50000  (511.80 per sec.)
    read/write requests:                 900000 (9212.48 per sec.)
    other operations:                    100000 (1023.61 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          97.6936s
    total number of events:              50000
    total time taken by event execution: 781.2354s
    response time:
         min:                                  5.55ms
         avg:                                 15.62ms
         max:                                234.14ms
         approx.  95 percentile:              24.31ms

Threads fairness:
    events (avg/stddev):           6250.0000/163.33
    execution time (avg/stddev):   97.6544/0.01

</code>
</pre>

## More work to do

So, like I said, these are just a bunch of notes I took when messing around with a virtual Galera cluster and doing some basic research into performance testing. I'll update this post as I continue on. Now that I have a virtualized test cluster that I can destroy and rebuild at will I can really get into understanding how it works and what the failure domains are, as well as how it performs. Eventually I would like to get [MariaDB MaxScale](https://mariadb.com/products/mariadb-maxscale) into the loop as well, and send writes to one host and reads to all.
