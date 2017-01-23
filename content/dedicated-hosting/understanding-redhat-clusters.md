---
permalink: understanding-redhat-clusters/
audit_date: ''
title: Dedicated Hosting FAQ
type: product
created_date: '2017-01-17'
created_by: Alan Hicks
last_modified_date: '2017-01-23'
last_modified_by: Alan Hicks
product: Dedicated Hosting
product_url: dedicated-hosting
---

### Overview

Rackspace supports high-availability linux clustering in our dedicated
environment. High-availability clusters offer the most robust environment
for your critical services by eliminating many single points of
failure. They are also valued for their ability to keep services online
during crucial maintenances. While both useful and beneficial, clusters
present unique requirements to implement and support them. Here we hope
to lay out the requirements, features, best practices, and potential
complications along with a brief discussion of some of the reasons you
may require or desire a high-availability cluster to help you decide if
this is the right decision for your business.

Some of the services we most commonly see implemented in clusters are:

- MySQL (including MariaDB and Percona)
- NFS
- Redis
- Oracle

### Requirements

As previously mentioned, proper implementation of a high-availability
cluster has requirements that are atypical of most servers at
Rackspace. 

- Operating Systems
    - Red Hat Linux version 5 is supported for legacy systems **only**. No new clusters will be deployed with RHEL 5.
    - Red Hat Linux version 6 is supported for new builds.
    - Red Hat Linux version 7 is not currently supported, but we are working diligently to add this offering.
    - Other Linux operation systems (e.g. Ubuntu, Debian) are **not** supported.
    - Customer provided operating systems (even those mentioned above as supported) are **not** supported.
- Hardware
    - Cloud servers are **not** supported.
    - New and legacy deployments supported on Dell R710, Dell R720, and HP GL380 servers.
    - Dell 2900 series servers are **not** supported.
- Shared Storage
    - Rackspace supports shared storage through our SAN and NAS offerings.
    - iSCSI and NFS storage solutions are also supported, but not recommended.
    - Rackspace highly recommends dedicated SAN or shared SAN for their performance benefits and robust nature.
    - GFS2 is **not** supported.

### Features

High availability clusters are usually desired for their ability to
keep crucial services such as databases and file shares online and
active in the event of unexpected server failures. When one member
experiences a failure resources and services homed to that member at
the time of the failure automatically migrate to another member.

Another facet of these clusters that rarely receives the attention it
deserves is the ability to take cluster members offline for maintenance
procedures without suffering a service outage. This enables maintenance
work to be performed during business hours without an interruption to
your core business activity. This is beloved by systems administrators
who are tired of waking up at 3 a.m. to perform some simple maintenance
that requires bringing a server offline. While Rackspace offers 24x7
support, many of our customers work regular 9-5 hours. HA clusters give
them the option to perform this type of operation during those regular
business hours without negatively impacting their customers.

### High-Availability Cabinets

In addition to making the servers themselves redundant, most of our
customers chose to place their clusters into our high-availability
cabinets. Doing so eliminates many other single points of failure by
adding the following features:

- Redundant Switches
    - Bonded network interfaces on the servers are connected to
      different switches, ensuring that a single switch or network
      card failure does not break access to the cluster.
- Redundant Firewalls
    - By placing your cluster behind our redundant firewalls, firewall
      maintenance or failure need not take your service offline.
- Redundant Power Circuits
    - All of our supported servers have redundant power supplies. By
      choosing our HA cabinet offering, these power supplies are
      connected to separate circuits, ensuring that the failure of a
      single electrical circuit need not cause an outage on your
      cluster.

### Fencing Broken Servers

Proper fencing is a requirement for a well-implemented HA cluster.
Fencing allows a working server to terminate a non-functional server,
forcing it to release its resources in the process. Fencing is
sometimes refered to as STONITH (Shoot The Other Node In The Head).

At Rackspace, fencing is handled by the servers' built-in out-of-bound
controller. On Dell devices, this is the DRAC module; HP servers use
their iLO module. As Dells are still the most prevalent servers at
Rackspace, we will refer to these devices as DRACs. Just keep in mind
that the actual controller module may be something slightly different.

The DRAC is an out-of-bound controller that allows administrators to
power servers on, power them off, or connect to the console over the
Internet, without logging into the server's operating system. Indeed,
an administrator can connect to the DRAC to perform operations even
when the server is shutdown or in a crashed state. Since we cannot rely
upon our ability to login to a failed cluster member via traditional
means, the DRAC is an ideal platform for fencing.

Should one of the cluster members crash or otherwise misbehave, its
peer will connect to the crashed member's DRAC and power cycle the
system. This is a robust and fast way to force a broken member to
release all of its resources immediately. The effects are not unlike
pressing the "reset" button on a workstation. All activity is
immediately terminated and the computer begins its BIOS checks.

Rebooting in this fashion not only gives the remaining cluster members
plenty of time to grab the offending node's resources, but it often has
the effect of fixing a great number of potential problems. For
instance, a very busy cluster member may run out of memory resources.
Those situations often cannot be fixed in any other way as the server
becomes fully non-responsive. It's also likely in some scenarios that a
cluster member could become I/O locked should its shared storage have
some sort of fault. Often the best (and sometimes the only) way to
clear these situations and release the shared storage device is to
reboot the entire server, as other members cannot utilize the shared
storage until the offending member releases it.

### Sharing Storage

Cluster members often need access to the same filesystems, so some form
of shared storage is a typical requirement. At Rackspace, we highly
encourage our customers to use either our dedicated or shared SAN
(Storage Attached Network) offerings, both for their performance
characteristics and their reliability. Our techs are trained to
understand and support these SAN devices on a daily basis, so our
familiarity with issues that can arise is high.

We also support storage through our NAS devices or other NFS offerings,
iscsi, and multipath. These additional options are less common however,
and typically do not perform at the same level as SAN.

Finally, Rackspace does not support methods of replicating filesystems
across cluster members. The most common tools for accomplishing this
are DRBD (Distributed Relational Block Device) and GFS2 (Global
Filesystem). In our internal tests, GFS2 has displayed many subtle and
unpredictable errors. Our analysis indicates that this technology is
simply not ready for the enterprise at this time. Similarly, DRBD can
silently fail in subtle ways. These situations are far less than ideal
for a robust clustered application, so we have elected not to support
them.

Rackspace uses LVM (Logical Volume Management) capabilities to share
storage between multiple members. Linux's LVM implementation includes a
cluster daemon (clvmd) which is responsible for locking filesystem to a
single member. When clvmd is active, it writes metadata to the volume
group to indicate which cluster member can access a logical volume at
any given time. This actively prevents a filesystem from being mounted
simultaneously by multiple machines and avoids filesystem corruption.

### Configuration

#### RHEL 5 and 6

In Red Hat versions 5 and 6, cluster configuration is largely handled
by the /etc/cluster/cluster.conf file. This is an XML file which
defines the members, fencing devices, clustered resources, and
clustered services. Let's take a look at an example. We won't go into
exhausting detail here, but a look at how things are defined may be
quite instructive.

##### Cluster Nodes

Every member of the cluster must have its own clusternode declaration.
This simply lists the node members, names them, and names their fencing
devices.

    <clusternodes> 
      <clusternode name="c80216-db1" votes="1"> 
        <fence> 
          <method name="1"> 
            <device name="db1"/> 
          </method> 
        </fence> 
      </clusternode> 
      <clusternode name="c80217-db2" votes="1"> 
        <fence> 
          <method name="1"> 
            <device name="db2"/> 
          </method> 
        </fence> 
      </clusternode> 
    </clusternodes> 

##### Fencing Devices

Every fence device is declared here. They are named and those names are
used as references for the node that would be fenced.

    <fencedevices> 
      <fencedevice agent="fence_ipmilan" auth="password" ipaddr="192.168.101.138" lanplus="1" login="root" name="db1" passwd="random" power_wait="4" delay="5"/>
      <fencedevice agent="fence_ipmilan" auth="password" ipaddr="192.168.101.139" lanplus="1" login="root" name="db2" passwd="random" power_wait="4"/>
    </fencedevices> 

The agent type here is "fence_ipmilan". This is the agent that
interfaces with DRAC devices and is used by Rackspace on all of our
supported clusters.

##### Resources

Resources must be defined individually. We will later group and order
these in service declarations.

    <resources> 
      <ip address="192.168.100.250" monitor_link="1"/> 
      <fs device="/dev/mapper/vgsan00-mysql00" force_unmount="1" fstype="ext3" mountpoint="/var/lib/mysql" name="MySQLData" options="defaults"/> 
      <script file="/etc/init.d/mysql" name="MySQL"/> 
      <ip address="10.241.36.250" monitor_link="0"/> 
    </resources> 

Here we have defined two floating IP addresses in the public subnet
(192.160.100.x) and two floating IP addresses in our servicenet subnet
(10.241.36.x) for backup operations. Additionally, we've created
resources to start a MySQL database and create an NFS export. Finally,
we've defined two different filesystem resources (fs) so that our
cluster can mount its shared storage devices on the proper member.

##### Services

Once resources have been defined, we can group them into different
services. These services ensure that every resource required to perform
a task (such as running MySQL) is grouped together. It also handles
starting and stopping those resources in the correct order to avoid
filesystem corruption.

Below, we've defined a database service entitled
"MySQL" which includes 4 different resources. By looking at the
inheritance of each XML declaration, we can see that the script
resource named "MySQL" is a child of the filesystem resource
"MySQLData". That in turn is dependent on the IP resource
192.168.100.250. Finally, this IP resource has a dependent IP resource
for 10.241.36.250. When this service is started, RCHS will first
provision the 192.168.100.250 IP address, then will provision the
10.241.36.250 resource as well as the MySQLData filesystem resource.
Once the filesystem resource has started, it will finally run the
script resource to start MySQL. Stopping the service works in the
reverse order.

    <service autostart="1" domain="cluster2" name="MySQL"> 
      <ip ref="192.168.100.250"> 
        <fs ref="MySQLData"> 
          <script ref="MySQL"/> 
        </fs> 
        <ip ref="10.241.36.250"/> 
      </ip> 
    </service> 

##### Complete cluster.conf

    <?xml version="1.0"?> 
    <cluster config_version="34" name="EXAMPLE"> 
      <fence_daemon clean_start="0" post_fail_delay="0" post_join_delay="3"/> 
      <clusternodes> 
        <clusternode name="c80216-db1" votes="1"> 
          <fence> 
            <method name="1"> 
              <device name="db1"/> 
            </method> 
          </fence> 
        </clusternode> 
        <clusternode name="c80217-db2" votes="1"> 
          <fence> 
            <method name="1"> 
              <device name="db2"/> 
            </method> 
          </fence> 
        </clusternode> 
      </clusternodes> 
      <cman expected_votes="1" two_node="1"/> 
      <fencedevices> 
        <fencedevice agent="fence_ipmilan" auth="password" ipaddr="192.168.101.138" lanplus="1" login="root" name="db1" passwd="random" power_wait="4" delay="5"/>
        <fencedevice agent="fence_ipmilan" auth="password" ipaddr="192.168.101.139" lanplus="1" login="root" name="db2" passwd="random" power_wait="4"/>
      </fencedevices> 
      <rm> 
        <failoverdomains> 
          <failoverdomain name="cluster1" ordered="0" restricted="0"> 
            <failoverdomainnode name="c80216-db1" priority="1"/> 
          </failoverdomain> 
          <failoverdomain name="cluster2" ordered="0" restricted="0"> 
            <failoverdomainnode name="c80217-db2" priority="1"/> 
          </failoverdomain> 
        </failoverdomains> 
        <resources> 
          <ip address="192.168.100.250" monitor_link="1"/> 
          <fs device="/dev/mapper/vgsan00-mysql00" force_unmount="1" fstype="ext3" mountpoint="/var/lib/mysql" name="MySQLData" options="defaults"/> 
          <script file="/etc/init.d/mysql" name="MySQL"/> 
          <ip address="10.241.36.250" monitor_link="0"/> 
        </resources> 
        <service autostart="1" domain="cluster2" name="MySQL"> 
          <ip ref="192.168.100.250"> 
            <fs ref="MySQLData"> 
              <script ref="MySQL"/> 
            </fs> 
            <ip ref="10.241.36.250"/> 
          </ip> 
        </service> 
      </rm> 
    </cluster>




