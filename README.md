# Highly Available Deployment of MySQL Router

## Introduction
MySQL Router can be deployed either with application servers or on its own mid-tier.

The advantage of deploying MySQL Router with the application is there is no need to worry about high availability: if the hosting machine is up and running correctly then, assuming MySQL Router has been configured and bootstrapped correctly, it will almost certainly continue to run. The disadvantage of deploying MySQL Router on application servers is that it has to be done for each and every application server. This is not too much of a problem if there is only a small numbers of application servers but becomes more onerous with large numbers (e.g. administration effort to both deploy and maintain). 

To overcome the issue of deploying MySQL Router on large numbers of application servers MySQL Router can be deployed on a separate mid-tier. This mid-tier needs to be made highly-available in order to prevent it representing a Single Point Of Failure (SPOF) which would largely defeat the point of making the backend database highly available. Currently (June 2020. MySQL 8.0.20 being the most recent release), MySQL Router has no built-in clustering function. Therefore to make a HA MySQL Router tier third party solutions are required.

One method, documented by MySQL, is to use DNS-SRV records in conjuction with software such as Consul and dnsmasq. In production this is difficult to implement and given it requires changes to DNS configurations which are likely to be resisted in production environments.

An alternative is to use standard clustering technologies and approaches. For example: software that manages cluster membership: automatically fails over resources when a node goes down, and which utilizes a floating IP address such that client software has a consistent point of attachment. The remainder of this document details how this can be achieved on Linux servers using Pacemaker and Corosync which collectively form a standard Linux clustering solution. This document has three main sections:

1. Environment Overview
2. Implementation of the HA MySQL Router Tier Solution
3. Testing Undertaking

In addition to these three sections there is an addendum which details a small amount of extra administration work in order to get the solution to work in the Oracle Cloud.

## Environment Overview
The diagram below details the environment that was built in order to firstly demonstrate how to setup and configure a highly-available clustered MySQL Router tier, and secondly to provide the necessary infrastructure to test the clustered tier.

![](../master/images/topology.png)

** MySQL Router Tier Stack **

A three node HA Cluster of MySQL Router nodes was created in the Oracle Cloud. Each node was an Oracle Compute Instance (i.e. a Virtual Machine) with the following stack:

  * Virtual machine specification: 1 core OCPU, 16 GB memory, 1 Gbps network bandwidth, nom 50GB storage
  * OS: Oracle Linux 7.8, kernel rev 4.14.35-1902.303.4.1.el7uek.x86_64
  * MySQL Router 8.0.20
  * MySQL Shell 8.0.20 (optional - used to test router connectivity)
  * Pacemaker 1.1.21
  
Notes: 
* Oracle Linux is a variant of Red Hat Enterprise Linux and as such it is expected that implementing a HA MySQL Router tier on either Red Hat, Centos or Fedora operating systems will work if configured in the same manner.
* The Ubuntu OS also has a Pacemaker implementation and so it is assumed that this will also work.
* Oracle Cloud: a small amount of additional integration work was required in order for the solution to work with Oracle Cloud's virtual network. This is detailed at the end of this document. It is anticipated that **no additional work** would be required with physical servers. Depending on how virtual networking is implemented in other cloud there may be some similar work required. 

## Implementation of the MySQL Router Tier Solution
This can be broken down into the following logical parts.
1. Software Stack Install
2. Security Considerations: firewalld, selinux and environmental
3. Bootstrapping the MySQL Router Tier 
4. Setup and Configuration of the Cluster

### Software Install of the MySQL Router Tier
Either the Enterprise or Community editions of MySQL software can be used. In the case below the RPM packages for MySQL are the commercial versions (enterprise edition) and were downloaded prior to install. Pacemaker and its associated packages are all available from Oracle-Linux/Redhat/Centos/Fedora repositories as standard.

Install the software as shown below on **all nodes** in the router tier.
```
% sudo yum localinstall --nogpgcheck mysql-router-commercial-8.0.20-1.1.el7.x86_64.rpm
% sudo yum localinstall --nogpgcheck mysql-shell-commercial-8.0.20-1.1.el7.x86_64.rpm
% sudo yum install pcs pacemaker resource-agents
```
For information: the install of the above software will see the creation of three accounts:
```
% tail -3 /etc/passwd
tss:x:59:59:Account used by the trousers package to sandbox the tcsd daemon:/dev/null:/sbin/nologin
hacluster:x:189:189:cluster user:/home/hacluster:/sbin/nologin
mysqlrouter:x:995:991:MySQL Router:/var/lib/mysqlrouter:/bin/false
%
```

### Security Considerations: firewalld, selinux and environmental
In order for application servers and other clients to connect to a MySQL Router the following ports need to be opened:
 * 6446/tcp - SQL protocol for read-write connections
 * 6447/tcp - SQL protocol for read-only connections
 * 64460/tcp - X protocol (for XDevAPI Document Store users) for read-write connections (this also allows SQL sessions)
 * 64470/tcp - X protocol (for XDevAPI Document Store users) for read-only connections (this also allows SQL sessions).
 
Pacemaker needs to communicate between the nodes using a variety of ports for both TCP and UDP protocols. Fortunately, Pacemaker is a well known package and the Linux firewall can be opened appropriately if the high-availability service is specified (using the service should also insulate you from any changes to ports that might come about as a result of an upgrade, etc. The list of ports that will be opened for Enterprise Linux 7 (RedHat, Centos, Oracle Linux can be found here: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/high_availability_add-on_reference/s1-firewalls-haar). 

On each node of the MySQL Router tier run the following commands:
```
% sudo firewall-cmd --zone=public --add-port=6446/tcp --permanent
% sudo firewall-cmd --zone=public --add-port=6447/tcp --permanent
% sudo firewall-cmd --zone=public --add-port=64460/tcp --permanent
% sudo firewall-cmd --zone=public --add-port=64470/tcp --permanent
% sudo firewall-cmd --permanent --add-service=high-availability
% sudo systemctl restart firewalld
% sudo firewall-cmd --list-ports
6446/tcp 64460/tcp 6447/tcp 64470/tcp
% sudo firewall-cmd --list-services
dhcpv6-client high-availability ssh
%
```
Note the last two commands confirm that the ports have been opened and the high-availability service is in place.

No change to the selinux mode (enforcing | permissive | disabled) is required because our cluster implementation is stateless (i.e. no need to write to storage).

If you are implementing in a Cloud Environment then you may need to make changes to your virtual networking. For example in the Oracle Cloud a rule had to be setup to allow TCP and UDP traffic to run on the virtual network that was being used.

**Naming Services**:

The test environment uses DNS (part of the Oracle Cloud) to resolve hostnames into IP addresses. If you don't have a naming service then you will have to enter the names and IP addresses of each router node and indeed each database node into /etc/hosts. Clients of the router tier will need to be made aware of the floating IP address.

### Bootstrapping of MySQL Router
To bootstrap the cluster we need to be aware of the backend database cluster or replica set being used. In this case a 3 node InnoDB Cluster is being used. It's Primary node is called ic1 and it's cluster administrator user account is clustadm@ic1. For each node in the MySQL Router cluster, run the following
```
% sudo mysqlrouter --user mysqlrouter --force --bootstrap clusteradm@ic1
% sudo systemctl restart mysqlrouter
```
**Test MySQL Router Instance Connectivity**:

On the **primary node** of the **InnoDB Cluster**:
 * create a document store schema and give it a collection
 * create a database and give it a table
 * create a user and grant that user select, insert, update and delete on both the schema and database. Note the user should be able to log in from anywhere.

For example:
```
% hostname
ic1
% mysqlsh
 MySQL  JS > \c root@localhost
 MySQL  localhost:33060+ ssl  JS > var schema = session.createSchema('ancestors')
 MySQL  localhost:33060+ ssl  JS > var col = schema.createCollection('flintstones')
 MySQL  localhost:33060+ ssl  JS > \sql
 MySQL  localhost:33060+ ssl  SQL > create database routertest;
 MySQL  localhost:33060+ ssl  SQL > use routertest;
 MySQL  localhost:33060+ ssl  routertest  SQL > create table t1 (
                                             -> id int auto_increment primary key,
                                             -> name char(20)
                                             -> );
 MySQL  localhost:33060+ ssl  routertest  SQL > create user stuart@'%' identified by 'MyPa55wd!';
 MySQL  localhost:33060+ ssl  routertest  SQL > grant select, insert, update, delete on routertest.* to stuart@'%';
 MySQL  localhost:33060+ ssl  routertest  SQL > grant select, insert, update, delete on ancestors.* to stuart@'%';
 MySQL  localhost:33060+ ssl  routertest  SQL > \q
%
```

On **each node** of the **MySQL Router tier**, perform these tests
```diff
- Using MySQL shell as a local client connect to the database using each of the MySQL Router ports: 6446 (SQL RW), 6447 (SQL RO), 64460 (X protocol RW), 66470 (X protocol RO)
% hostname
rt1
% mysqlsh --uri stuart@localhost:6446         # Change port to 6447, 64460 and 64470 as required
 Please provide the password for 'stuart@localhost:6446': *********
 MySQL  localhost:6446 ssl  JS >

- Test 1: Toggle to SQL and check which node of the InnoDB Cluster we are connected to. 
-         For connections on ports 6446 and 64460 we should be on the Primary node (RW)
-         For connections on ports 6447 and 64470 we should be on one of the Secondary nodes (RO)
 MySQL  localhost:6446 ssl  SQL > select @@hostname;
 +------------+
 | @@hostname |
 +------------+
 | ic1        |
 +------------+
 1 row in set (0.0002 sec)
 MySQL  localhost:6446 ssl  SQL >
 
- Test 2: Toggle to JavaScript and check whether you can connect to Document Store.
-         Only connections on the X Protocol ports 64460 and 64470 should be allowed connections
-         Connections on classic SQL ports (6446 and 6447) will receive error messages
 MySQL  localhost:6446 ssl  SQL > \js
 Switching to JavaScript mode...
 MySQL  localhost:6446 ssl  JS > var schema = session.getSchema("ancestors")
 Invalid object member getSchema (AttributeError)
 MySQL  localhost:6446 ssl  JS >
 
- Test 3: If the schema object was obtained in test 2, access its collection and then add a document to it.
-         This cannot be done for connections on classic SQL ports (6446 and 6447) given they won't have been able to create the schema object.
-         This test will work in its entirety for connections on 64460 because they are read-write. 
-         Connections on port 64470 will only be able to do the query part (i.e. the find()) because connections on this port are read-only
 MySQL  localhost:64460+ ssl  JS > var collection = schema.getCollection("flintstones")
 MySQL  localhost:64460+ ssl  JS > collection.add({"name": "Fred", "type": "Early Human"})
 Query OK, 1 item affected (0.0062 sec)
 MySQL  localhost:64460+ ssl  JS > collection.find()
 {
     "_id": "00005ef4b9130000000000000001",
     "name": "Fred",
     "type": "Early Human"
 }
 1 document in set (0.0005 sec)
 MySQL  localhost:64460+ ssl  JS >

- Test 4: Toggle to SQL mode and use the routertest database
-         This will work for all port types
 MySQL  localhost:6446 ssl  JS > \sql
 MySQL  localhost:6446 ssl  SQL > use routertest;
 Default schema set to `routertest`.
 Fetching table and column names from `routertest` for auto-completion... Press ^C to stop.
 MySQL  localhost:6446 ssl  routertest  SQL >
 
- Test 5: Insert a row to routertest's table, then query it
-         This test will work in its entirety for connections using ports 6446 and 64460 because they are read-write
-         Connections using ports 6447 and 64470 will not be able to do the insert but will be able to do the query because they are read-only
 MySQL  localhost:6446 ssl  routertest  SQL > insert into t1 (name) values ("Stuart");
 Query OK, 1 row affected (0.0058 sec)
 MySQL  localhost:6446 ssl  routertest  SQL > select * from t1;
 +----+--------+
 | id | name   |
 +----+--------+
 |  1 | stuart |
 +----+--------+
 1 row in set (0.0005 sec)
 MySQL  localhost:6446 ssl  routertest  SQL > \q

- Now repeat the above until all MySQL Router nodes and ports have been tested.
```

## Setup and Configuration of the Cluster

**Initial Setup of the Pacemaker Cluster**:

Set the password for the hacluster user account **on each of the MySQL Router nodes**:
```
% sudo passwd hacluster
New password: 
Retype new password:
passwd: all authentication tokens updated successfully.
```
In the example above the password used was MyPa55wd! - this will be use when we come to create the cluster.

Now start the pcsd service **on each MySQL Router node** in order that we can create the cluster. Assuming the start is successful then enable the service so that it automatically restarts on a reboot. 
```
% sudo systemctl start pcsd.service
% sudo systemctl enable pcsd.service
```

With the pcsd service now running on each node, we can create the cluster. Firstly, set up authentication between the nodes using the hacluster user, then create the cluster and finally start it. On **one node of the cluster**:
```
% sudo pcs cluster auth rt1 rt2 rt3 -u hacluster -p MyPa55wd!
rt1: Authorized
rt2: Authorized
rt3: Authorized
% sudo pcs cluster setup --name mysqlroutercluster rt1 rt2 rt3    
Destroying cluster on nodes: mrt1, mrt2...
rt1: Stopping Cluster (pacemaker)...
rt2: Stopping Cluster (pacemaker)...
rt3: Stopping Cluster (pacemaker)...
rt1: Successfully destroyed cluster
rt2: Successfully destroyed cluster
rt3: Successfully destroyed cluster

Sending 'pacemaker_remote authkey' to 'mrt1', 'mrt2'
rt1: successful distribution of the file 'pacemaker_remote authkey'
rt2: successful distribution of the file 'pacemaker_remote authkey'
rt3: successful distribution of the file 'pacemaker_remote authkey'
Sending cluster config files to the nodes...
rt1: Succeeded
rt2: Succeeded
rt3: Succeeded

Synchronizing pcsd certificates on nodes mrt1, mrt2...
rt1: Success
rt2: Success
rt3: Success
Restarting pcsd on the nodes in order to reload the certificates...
rt1: Success
rt2: Success
rt3: Success

% sudo pcs cluster start --all
rt1: Starting Cluster (corosync)...
rt2: Starting Cluster (corosync)...
rt3: Starting Cluster (corosync)...
rt1: Starting Cluster (pacemaker)...
rt2: Starting Cluster (pacemaker)...
rt3: Starting Cluster (pacemaker)...
%
```
Give the cluster ~30 seconds to establish itself and then check its status. All the nodes must be online. 
```
% sudo pcs status
Cluster name: mysqlroutercluster

WARNINGS:
No stonith devices and stonith-enabled is not false

Stack: corosync
Current DC: rt1 (version 1.1.21-4.el7-f14e36fd43) - partition with quorum
Last updated: Thu Jul  2 13:51:38 2020
Last change: Thu Jul  2 13:51:38 2020 by hacluster via crmd on rt1

3 nodes configured
0 resources configured

Online: [ rt1 rt2 rt3 ]

No resources


Daemon Status:
  corosync: active/disabled
  pacemaker: active/disabled
  pcsd: active/enabled
%
```
Some points to note:
* The cluster can be monitored from any node using either the status command as shown or crm_mon. If crm_mon is run then it will act as a console and continue to be updated. An alternative usage is to specify the number of times you want it to sample before exiting. For example to take a snapshot you would specify "one" as follows: crm_mon -1
* From the above status output we can see that 
  * We have warnings with respect to stonith (Shoot The Other Node In The Head - used for split brain) and these will need to be addressed. 
  * Quorum is in place, however, for our cluster we need to change the default behaviour when quorum is lost. 
  * We have a cluster but no resources are configured. Configuring resources (i.e. providing a VIP, associating the MySQL Router service with the cluster) will be done as one of the next steps.
  * We can also see that both pacemaker and corosync daemons are active/disabled. All this means is that these daemons are running (under systemd) but they have not been enabled to allow systemd to restart them upon reboot, etc.

**Configure Pacemaker Properties**:

```
% sudo pcs property set no-quorum-policy=ignore
% sudo pcs property set stonith-enabled=false
% sudo resource defaults migration-threshold=1
```

**Assigning Resources to the Pacemaker Cluster**:

```
% sudo pcs resource create Router_VIP ocf:heartbeat:IPaddr2 ip=10.0.0.101 cidr_netmask=16 nic=ens3 op monitor interval=5s
% sudo pcs resource create mysqlrouter systemd:mysqlrouter clone
% sudo pcs constraint colocation add Router_VIP with mysqlrouter-clone score=INFINITY
```

**Readying the Cluster for Testing**:

```
% sudo pcs cluster stop --all
% sudo pcs cluster start --all
```
Note: if you are deploying on the Oracle Cloud

## Testing

## Additional Work Required for the Oracle Cloud
**Problem statement**: when the active node fails over to a passive node, the floating IP address must be moved to this passive node in order for it to become the new active node. Pacemaker understands that this is required but Oracle virtual networking is reluctant to reassign the floating IP address to the new active node. This understandable because in normal circumstances it is not desirable to have the potential of two or more interfaces on the same network using the same IP address.

**Solution**: explicitly instruct Oracle virtual network to remove the floating IP address from the failed node and reassign it to the new active node. The Pacemaker stack install provides an Open Cluster Framework Resource Agent script file, /usr/lib/ocf/resource.d/heartbeat/IPaddr2, whose purpose is to provide an interface to manage IP resources. 

**Implementation**: install Oracle Cloud Infrastructure (OCI) Command Line Interface (CLI) utility on each node in order to provide a script interface to the Oracle Cloud so that when a failover event triggers the script file, /usr/lib/ocf/resource.d/heartbeat/IPaddr2, we can make a call to OCI that will reassign the IP address.

To install and configure the OCI CLI follow the procedure detailed here: https://docs.cloud.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm

Before you can edit /usr/lib/ocf/resource.d/heartbeat/IPaddr2 you will need to obtain the following information:
1. The name of the VNIC which will be used to host the floating IP address. This is typically ens3, but you should check using ip addr (on each node in the cluster!).
2. The OCID of the VNIC. Log into the OCI Console, then for each node navigate to VNIC Details (Compute > Instances > Instance Details > Attached VNICs > VNIC Details). Copy the VNIC's OCID value.

Once installed and configured you will need to add the following lines of code to /usr/lib/ocf/resource.d/heartbeat/IPaddr2. These lines were added immediately under the header comments (line 65 to be precise):
```sh
##### OCI vNIC variables
SERVER=`hostname -s`
RT1_VNIC="ocid1.vnic.oc1.uk-london-1.abwgiljsbwszs6e....................................3qgawid6q"
RT2_VNIC="ocid1.vnic.oc1.uk-london-1.abwgiljschdlc72....................................7gga2xunq"
RT3_VNIC="ocid1.vnic.oc1.uk-london-1.abwgiljsxjhqmiw....................................p7tt22iwa"
FLOATING_IP="10.0.0.101"
##### OCI/IPaddr Integration
if [ $SERVER = "rt1" ]; then
        /root/bin/oci network vnic assign-private-ip --unassign-if-already-assigned --vnic-id $RT1_VNIC --ip-address $FLOATING_IP
elif [ $SERVER = "rt2" ]; then
        /root/bin/oci network vnic assign-private-ip --unassign-if-already-assigned --vnic-id $RT2_VNIC --ip-address $FLOATING_IP
elif [ $SERVER = "rt3" ]; then
        /root/bin/oci network vnic assign-private-ip --unassign-if-already-assigned --vnic-id $RT3_VNIC --ip-address $FLOATING_IP
fi
```

The same code can be written to /usr/lib/ocf/resource.d/heartbeat/IPaddr2 across all nodes without change. However, much of the code will never be accessed because the if statement on each server will always select just one line of code to be executed. As such, all that is needed is the /root/bin/oci... line with the requisite vnic-id value for the node it is being executed on.

**Further Issue**: the above OCI CLI / IPaddr2 solution worked as detailed above. However, on a subsequent implementation with a slightly later kernel implementation and newer version of OCI CLI (old version 2.12.0, new 2.12.1) it ceased to work. The problem was traced to Python 3 not being happy with the locale. The fix to this issue was to explicitly set the locale in the /usr/lib/ocf/resource.d/heartbeat/IPaddr2 script. At line 65:
```sh
export LC_ALL=C.UTF-8
export LANG=C.UTF-8
##### OCI vNIC variables
SERVER=`hostname -s`
```
Once this was done, normal service was resumed.
