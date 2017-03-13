---
permalink: understanding-redhat-clusters/
audit_date: ''
title: Dedicated Hosting FAQ
type: product
created_date: '2017-01-17'
created_by: Alan Hicks
last_modified_date: '2017-03-13'
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
    - GFS2, glusterfs, and similar filesystems are **not** supported.

### Features

High availability clusters are usually desired for their ability to
keep crucial services such as databases and file shares online and
active in the event of unexpected server failures. When one member
experiences a failure, resources and services homed to that member at
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
business hours without negatively impacting their operations.

### High-Availability Cabinets

In addition to making the servers themselves redundant, most of our
customers choose to place their clusters into our high-availability
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

At Rackspace, fencing is handled by the servers' built-in out-of-band
controller. On Dell devices, this is the DRAC module; HP servers use
their iLO module. As Dells are still the most prevalent servers at
Rackspace, we will refer to these devices as DRACs. Just keep in mind
that the actual controller module may be something slightly different.

The DRAC is an out-of-band controller that allows administrators to
power servers on, power them off, or connect to the console over the
network without logging into the server's operating system. Indeed,
an administrator can connect to the DRAC to perform operations even
when the server is shutdown or in a crashed state. Since we cannot rely
upon our ability to login to a failed cluster member via traditional
means, the DRAC is an ideal platform for fencing.

Should one of the cluster members crash or otherwise misbehave, its
peer will connect to the failed member's DRAC and power cycle the
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
cluster daemon (clvmd) which is responsible for locking filesystems to a
single member. When clvmd is active, it writes metadata to the volume
group to indicate which cluster member can access a logical volume at
any given time. This actively prevents a filesystem from being mounted
simultaneously by multiple machines and avoids filesystem corruption.

### RHEL 5 and 6

In Red Hat versions 5 and 6, cluster configuration is largely handled
by the /etc/cluster/cluster.conf file. This is an XML file which
defines the members, fencing devices, clustered resources, and
clustered services. Let's take a look at an example. We won't go into
exhaustive detail here, but a look at how things are defined may be
quite instructive.

#### Cluster Nodes

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

#### Fencing Devices

Every fence device is declared here. They are named and those names are
used as references for the node that would be fenced.

    <fencedevices> 
      <fencedevice agent="fence_ipmilan" auth="password" ipaddr="192.168.101.138" lanplus="1" login="root" name="db1" passwd="random" power_wait="4" delay="5"/>
      <fencedevice agent="fence_ipmilan" auth="password" ipaddr="192.168.101.139" lanplus="1" login="root" name="db2" passwd="random" power_wait="4"/>
    </fencedevices> 

The agent type here is "fence_ipmilan". This is the agent that
interfaces with DRAC devices and is used by Rackspace on all of our
supported clusters.

#### Resources

Resources must be defined individually. We will later group and order
these in service declarations.

    <resources> 
      <ip address="192.168.100.250" monitor_link="1"/> 
      <fs device="/dev/mapper/vgsan00-mysql00" force_unmount="1" fstype="ext3" mountpoint="/var/lib/mysql" name="MySQLData" options="defaults"/> 
      <script file="/etc/init.d/mysql" name="MySQL"/> 
      <ip address="10.241.36.250" monitor_link="0"/> 
    </resources> 

Here we have defined a floating IP address in the public subnet
(192.160.100.250) and a floating IP address in our servicenet subnet
(10.241.36.250) for backup operations. Additionally, we've created a
filesystem resource (fs) so our cluster can mount its shared storage
device. Finally, we have a script resource to start the MySQL daemon.

#### Services

Once resources have been defined, we can group them into different
services. These services ensure that every resource required to perform
a task (such as running MySQL) is grouped together. It also handles
starting and stopping those resources in the correct order.

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

#### Complete cluster.conf

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

#### Cluster Status

Viewing the current status of your cluster is easily accomplished with
**clustat**.

    # clustat
    Cluster Status for mysql-clus
    Member Status: Quorate
    
     Member Name            ID   Status
     ------ ----            ---- ------
     c80216-db1             1 Online, Local, rgmanager
     c80217-db2             2 Online, rgmanager
    
     Service Name           Owner (Last)                 State
     ------- ----           ----- ------                 -----
     service:mysql-svc      c80216-db1                   started

As you can see here, both c80216-db1 and c80217-db2 are online. There is
only a single service defined (mysql-svc) and it is currently running on
c80216-db1.

#### Cluster Administration

**clusvcadm** is used to restart services, move services, and
enable/disable services. On all clusters Rackspace creates, the
**/etc/motd** file is edited to include a self-explainatory message
that will assist you in performing several common tasks:

    'clusvcadm -R mysqlsvc' ~= Will restart MySQL in place on the same server
    'clusvcadm -r mysqlsvc -m <node name>' ~= Will relocate MySQL to that node
    'clusvcadm -d mysqlsvc' ~= Will disable MySQL
    'clusvcadm -e mysqlsvc' ~= Will enable MySQL


### RHEL 7

In version 7, Red Hat replaced its older clustering software with new
alternatives. Some of the old daemons still exist, but configuration,
monitoring, and management are handled by a new python utility called
**pcs**.

#### Cluster Nodes

Every member of the cluster must have its own clusternode declaration.
This simply lists the node members, names them, and names their fencing
devices.

    # pcs config show | head -n 5
    Cluster Name: pcs2016-11-01
    Corosync Nodes:
     256188-linclus2a 256189-linclus2b 
    Pacemaker Nodes:
     256188-linclus2a 256189-linclus2b

#### Fencing

Declaring fencing methods operates a bit differently in RHEL 7.

    # pcs config show | grep -A 6 ^Stonith
    Stonith Devices: 
     Resource: fence_node1 (class=stonith type=fence_ipmilan)
      Attributes: power_wait=4 ipaddr=192.168.100.20 action=reboot login=root lanplus=1 pcmk_host_list=256188-linclus2a pcmk_host_check=static-list passwd=yMkxYVKPByE9zRN 
      Operations: monitor interval=60s (fence_node1-monitor-interval-60s)
     Resource: fence_node2 (class=stonith type=fence_ipmilan)
      Attributes: power_wait=4 ipaddr=192.168.100.21 action=reboot login=root lanplus=1 pcmk_host_list=256189-linclus2b pcmk_host_check=static-list passwd=kIZC8UxS846Dujn delay=4 
      Operations: monitor interval=60s (fence_node2-monitor-interval-60s)

#### Services and Resources

Unlike RHEL 5/6 where resources are defined individually, then added to
a service, in RHEL 7 a resource group is defined and then resources are
added to it. This creates a single resource group which can be stopped
and started on demand.

    # pcs resource show mysql-svc
     Group: mysql-svc
      Meta Attrs: migration-threshold=2 failure-timeout=60s 
      Resource: mysql_ip (class=ocf provider=heartbeat type=IPaddr2)
       Attributes: ip=192.168.100.12 
       Operations: start interval=0s timeout=20s (mysql_ip-start-interval-0s)
         stop interval=0s timeout=20s (mysql_ip-stop-interval-0s)
         monitor interval=10s timeout=10s on-fail=restart (mysql_ip-monitor-interval-10s)
      Resource: mysql_lvm (class=ocf provider=heartbeat type=LVM)
       Attributes: volgrpname=vgmysql00 exclusive=true 
       Operations: start interval=0s timeout=30 (mysql_lvm-start-interval-0s)
         stop interval=0s timeout=30 (mysql_lvm-stop-interval-0s)
         monitor interval=10s timeout=10s on-fail=restart (mysql_lvm-monitor-interval-10s)
      Resource: mysql_fs (class=ocf provider=heartbeat type=Filesystem)
       Attributes: device=/dev/vgmysql00/lvmysql00 directory=/san/mysql-fs fstype=ext4 options=nobarrier 
       Operations: start interval=0s timeout=60 (mysql_fs-start-interval-0s)
         stop interval=0s timeout=60 (mysql_fs-stop-interval-0s)
         monitor interval=10s timeout=10s on-fail=restart (mysql_fs-monitor-interval-10s)
      Resource: mysql_srv (class=ocf provider=heartbeat type=mysql)
       Attributes: config=/etc/my.cnf enable_creation=0 user=mysql group=mysql datadir=/san/mysql-fs/mysql socket=/san/mysql-fs/mysql/mysql.sock additional_parameters=--bind-address=192.168.100.12 
       Operations: promote interval=0s timeout=120 (mysql_srv-promote-interval-0s)
         demote interval=0s timeout=120 (mysql_srv-demote-interval-0s)
         monitor interval=10s timeout=10s on-fail=restart (mysql_srv-monitor-interval-10s)
         start interval=0 timeout=600s (mysql_srv-start-interval-0)
         stop interval=0 timeout=600s (mysql_srv-stop-interval-0)
      Resource: mysql_snet_up (class=ocf provider=heartbeat type=IPaddr2)
       Attributes: ip=10.241.177.133 
       Operations: start interval=0s timeout=20s (mysql_snet_up-start-interval-0s)
         stop interval=0s timeout=20s (mysql_snet_up-stop-interval-0s)
         monitor interval=10s timeout=10s on-fail=ignore (mysql_snet_up-monitor-interval-10s)

#### Location Constraints

Often it's important that services don't run on a particular member of
the cluster, or that two services run together on the same member (or
alternatively *never* run on the same member). This is accomplished by
configuring resource constraints. In particular, you will always want
to ensure that the fencing resource **never** runs on the member it
fences. If this happens, then it is impossible to properly fence that
device in many failure scenarios.

    # pcs config show | grep -A 10 ^Location
    Location Constraints:
      Resource: fence_node1
        Disabled on: 256188-linclus2a (score:-INFINITY) (id:location-fence_node1-256188-linclus2a--INFINITY)
      Resource: fence_node2
        Disabled on: 256189-linclus2b (score:-INFINITY) (id:location-fence_node2-256189-linclus2b--INFINITY)
      Resource: mysql-svc
        Enabled on: 256189-linclus2b (score:INFINITY) (role: Started) (id:cli-prefer-mysql-svc)
    Ordering Constraints:
      start dlm-clone then start clvmd-clone (kind:Mandatory) (id:order-dlm-clone-clvmd-clone-mandatory)
      start clvmd-clone then start mysql-svc (kind:Mandatory) (id:order-clvmd-clone-mysql-svc-mandatory)
    Colocation Constraints:

#### Complete Configuration

    # pcs config show
    Cluster Name: pcs2016-11-01
    Corosync Nodes:
     256188-linclus2a 256189-linclus2b 
    Pacemaker Nodes:
     256188-linclus2a 256189-linclus2b 
    
    Resources: 
     Clone: dlm-clone
      Meta Attrs: notify=true interleave=true ordered=false 
      Resource: dlm (class=ocf provider=pacemaker type=controld)
       Operations: start interval=0s timeout=90 (dlm-start-interval-0s)
         stop interval=0s timeout=100 (dlm-stop-interval-0s)
         monitor interval=60s timeout=60s on-fail=fence (dlm-monitor-interval-60s)
     Clone: clvmd-clone
      Meta Attrs: notify=true interleave=true ordered=false 
      Resource: clvmd (class=ocf provider=heartbeat type=clvm)
       Attributes: activate_vgs=false 
       Operations: start interval=0s timeout=90 (clvmd-start-interval-0s)
         stop interval=0s timeout=90 (clvmd-stop-interval-0s)
         monitor interval=60s timeout=60s on-fail=fence (clvmd-monitor-interval-60s)
     Group: mysql-svc
      Meta Attrs: migration-threshold=2 failure-timeout=60s 
      Resource: mysql_ip (class=ocf provider=heartbeat type=IPaddr2)
       Attributes: ip=192.168.100.12 
       Operations: start interval=0s timeout=20s (mysql_ip-start-interval-0s)
         stop interval=0s timeout=20s (mysql_ip-stop-interval-0s)
         monitor interval=10s timeout=10s on-fail=restart (mysql_ip-monitor-interval-10s)
      Resource: mysql_lvm (class=ocf provider=heartbeat type=LVM)
       Attributes: volgrpname=vgmysql00 exclusive=true 
       Operations: start interval=0s timeout=30 (mysql_lvm-start-interval-0s)
         stop interval=0s timeout=30 (mysql_lvm-stop-interval-0s)
         monitor interval=10s timeout=10s on-fail=restart (mysql_lvm-monitor-interval-10s)
      Resource: mysql_fs (class=ocf provider=heartbeat type=Filesystem)
       Attributes: device=/dev/vgmysql00/lvmysql00 directory=/san/mysql-fs fstype=ext4 options=nobarrier 
       Operations: start interval=0s timeout=60 (mysql_fs-start-interval-0s)
         stop interval=0s timeout=60 (mysql_fs-stop-interval-0s)
         monitor interval=10s timeout=10s on-fail=restart (mysql_fs-monitor-interval-10s)
      Resource: mysql_srv (class=ocf provider=heartbeat type=mysql)
       Attributes: config=/etc/my.cnf enable_creation=0 user=mysql group=mysql datadir=/san/mysql-fs/mysql socket=/san/mysql-fs/mysql/mysql.sock additional_parameters=--bind-address=192.168.100.12 
       Operations: promote interval=0s timeout=120 (mysql_srv-promote-interval-0s)
         demote interval=0s timeout=120 (mysql_srv-demote-interval-0s)
         monitor interval=10s timeout=10s on-fail=restart (mysql_srv-monitor-interval-10s)
         start interval=0 timeout=600s (mysql_srv-start-interval-0)
         stop interval=0 timeout=600s (mysql_srv-stop-interval-0)
      Resource: mysql_snet_up (class=ocf provider=heartbeat type=IPaddr2)
       Attributes: ip=10.241.177.133 
       Operations: start interval=0s timeout=20s (mysql_snet_up-start-interval-0s)
         stop interval=0s timeout=20s (mysql_snet_up-stop-interval-0s)
         monitor interval=10s timeout=10s on-fail=ignore (mysql_snet_up-monitor-interval-10s)
     Clone: Pacewatcher-clone
      Meta Attrs: interleave=true ordered=false 
      Resource: Pacewatcher (class=ocf provider=pacemaker type=ClusterMon)
       Attributes: user=root update=10000 extra_options="-E /usr/local/sbin/pacewatcher --watch-fencing" pidfile=/var/run/pacewatcher.pid 
       Operations: start interval=0s timeout=20 (Pacewatcher-start-interval-0s)
         stop interval=0s timeout=20 (Pacewatcher-stop-interval-0s)
         monitor interval=10 timeout=20 (Pacewatcher-monitor-interval-10)
    
    Stonith Devices: 
     Resource: fence_node1 (class=stonith type=fence_ipmilan)
      Attributes: power_wait=4 ipaddr=192.168.100.20 action=reboot login=root lanplus=1 pcmk_host_list=256188-linclus2a pcmk_host_check=static-list passwd=yMkxYVKPByE9zRN 
      Operations: monitor interval=60s (fence_node1-monitor-interval-60s)
     Resource: fence_node2 (class=stonith type=fence_ipmilan)
      Attributes: power_wait=4 ipaddr=192.168.100.21 action=reboot login=root lanplus=1 pcmk_host_list=256189-linclus2b pcmk_host_check=static-list passwd=kIZC8UxS846Dujn delay=4 
      Operations: monitor interval=60s (fence_node2-monitor-interval-60s)
    Fencing Levels: 
    
    Location Constraints:
      Resource: fence_node1
        Disabled on: 256188-linclus2a (score:-INFINITY) (id:location-fence_node1-256188-linclus2a--INFINITY)
      Resource: fence_node2
        Disabled on: 256189-linclus2b (score:-INFINITY) (id:location-fence_node2-256189-linclus2b--INFINITY)
      Resource: mysql-svc
        Enabled on: 256189-linclus2b (score:INFINITY) (role: Started) (id:cli-prefer-mysql-svc)
    Ordering Constraints:
      start dlm-clone then start clvmd-clone (kind:Mandatory) (id:order-dlm-clone-clvmd-clone-mandatory)
      start clvmd-clone then start mysql-svc (kind:Mandatory) (id:order-clvmd-clone-mysql-svc-mandatory)
    Colocation Constraints:
    
    Resources Defaults:
     No defaults set
    Operations Defaults:
     No defaults set
    
    Cluster Properties:
     cluster-infrastructure: corosync
     cluster-name: pcs2016-11-01
     dc-version: 1.1.13-10.el7_2.4-44eb2dd
     default-resource-stickiness: 100
     have-watchdog: false

#### Cluster Status

Viewing the current status for the cluster is likewise accomplished with
pcs.

    # pcs status 
    Cluster name: pcs2016-11-01
    Last updated: Mon Jan 23 15:08:19 2017		Last change: Thu Jan 19 17:14:50 2017 by root via crm_resource on 256189-linclus2b
    Stack: corosync
    Current DC: 256188-linclus2a (version 1.1.13-10.el7_2.4-44eb2dd) - partition with quorum
    2 nodes and 13 resources configured
    
    Online: [ 256188-linclus2a 256189-linclus2b ]
    
    Full list of resources:
    
     fence_node1	(stonith:fence_ipmilan):	Started 256189-linclus2b
     fence_node2	(stonith:fence_ipmilan):	Started 256188-linclus2a
     Clone Set: dlm-clone [dlm]
         Started: [ 256188-linclus2a 256189-linclus2b ]
     Clone Set: clvmd-clone [clvmd]
         Started: [ 256188-linclus2a 256189-linclus2b ]
     Resource Group: mysql-svc
         mysql_ip	(ocf::heartbeat:IPaddr2):	Started 256189-linclus2b
         mysql_lvm	(ocf::heartbeat:LVM):	Started 256189-linclus2b
         mysql_fs	(ocf::heartbeat:Filesystem):	Started 256189-linclus2b
         mysql_srv	(ocf::heartbeat:mysql):	Started 256189-linclus2b
         mysql_snet_up	(ocf::heartbeat:IPaddr2):	Started 256189-linclus2b
     Clone Set: Pacewatcher-clone [Pacewatcher]
         Started: [ 256188-linclus2a 256189-linclus2b ]
    
    PCSD Status:
      256188-linclus2a: Online
      256189-linclus2b: Online
    
    Daemon Status:
      corosync: active/enabled
      pacemaker: active/enabled
      pcsd: active/enabled

#### Cluster Administration

Similarly, pcs is used to administer the cluster. On all clusters
Rackspace creates, the /etc/motd file is edited to include a
self-explainatory message like:

    'pcs cluster standby <node>'      ~= Place <node> into Standby
    'pcs cluster unstandby <node>'    ~= Take <node> out of Standby
    
    'pcs resource disable my-svc'     ~= Disable resource group my-svc
    'pcs resource enable my-svc'     ~= Enable resource group my-svc
    'pcs resource restart my-svc'     ~= Restart resource group my-svc
    'pcs resource move my-svc <node>' ~= Move my-svc to <node> + create a constraint
    'pcs resource clear my-svc'       ~= Clear resource constraints after a move



