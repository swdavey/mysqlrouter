# Highly Available Deployment of MySQL Router

## Introduction
MySQL Router can be deployed either with indidvidual application servers or on its own mid-tier.

The advantage of deploying MySQL Router with application servers is there is no need to worry about high availability: if the hosting machine is up and running correctly then, assuming MySQL Router has been configured and bootstrapped correctly, it will almost certainly continue to run. The disadvantage of deploying MySQL Router on application servers is that it has to be done for each and every application server. This is not too much of a problem if there is only a small numbers of application servers but it becomes onerous with large numbers (i.e. administration effort to both deploy and maintain). 

To overcome the issue of deploying MySQL Router on large numbers of application servers MySQL Router can be deployed on a separate mid-tier. This mid-tier needs to be made highly-available in order to prevent it representing a Single Point Of Failure (SPOF) which would defeat the point of making the backend database highly available. Currently (June 2020. MySQL 8.0.20 being the most recent release), MySQL Router has no built-in clustering function. Therefore to make a Highly Available MySQL Router tier third party solutions are required.

One method, documented by MySQL, is to use DNS-SRV records in conjuction with software such as Consul and dnsmasq. In production this is difficult to implement because it requires changes to DNS configurations and these are often resisted by network administrators.

An alternative is to use standard clustering technologies and approaches. For example, software that: manages cluster membership; automatically fails over resources on error, and which utilizes a floating IP address such that clients have a consistent point of attachment. The remainder of this document details how this can be achieved on Linux servers using Pacemaker and Corosync which collectively form a standard Linux clustering solution. This document has three main sections:

1. Environment Overview
2. Implementation of the HA MySQL Router Tier Solution
3. Testing Undertaking

In addition to these three sections there is an addendum which details a small amount of extra administration work in order to get the solution to work in the Oracle Cloud.

## Environment Overview
The diagram below details the environment that was built in the Oracle Cloud to demonstrate how to setup and configure a highly-available clustered MySQL Router tier as well as to test the same.

![](../master/images/topology.png)

**MySQL Router Tier stack:**

A three node HA Cluster of MySQL Router nodes was created in the Oracle Cloud. Each node was an Oracle Compute Instance (i.e. a Virtual Machine) with the following stack:

  * Virtual machine specification: 1 core OCPU, 16 GB memory, 1 Gbps network bandwidth, nom 50GB storage
  * OS: Oracle Linux 7.8, kernel rev 4.14.35-1902.303.4.1.el7uek.x86_64
  * MySQL Router 8.0.20
  * MySQL Shell 8.0.20 (optional - used to test router connectivity)
  * Pacemaker HA Resource Manager 1.1.21
  * Corosync Cluster Engine 2.4.5
  
Notes: 
* Oracle Linux is a variant of Red Hat Enterprise Linux and as such it is expected that implementing a HA MySQL Router tier on either Red Hat, Centos or Fedora operating systems will work if configured in the same manner.
* The Ubuntu OS also has a Pacemaker implementation and so it is assumed that this will also work.
* Pacemaker and Corosync collectively work together to provide a highly available clustering solution which is managed through its **pcs** interface (Pacemaker/Corosync Configuration System). Given there are too many three lettered acronyms in the world and typing/reading Pacemaker and Corosync is hardwork, the cluster in this document will be referred to as Pacemaker. 
* Oracle Cloud: a small amount of additional integration work was required in order for the solution to work with Oracle Cloud's virtual network. This is detailed at the end of this document. It is anticipated that **no additional work** would be required with physical servers. Depending on how virtual networking is implemented in other cloud there may be some similar work required. 

## Implementation of the MySQL Router Tier Solution
This can be broken down into the following logical parts.
1. Software Stack Install
2. Security Considerations: firewalld, selinux and environmental
3. Bootstrapping the MySQL Router Tier 
4. Setup and Configuration of the Cluster

### Software Stack Install of the MySQL Router Tier
Either the Enterprise or Community editions of MySQL software can be used. In the case below the RPM packages for MySQL are the commercial versions (enterprise edition) and were downloaded prior to install. Pacemaker and its associated packages are all available from Oracle-Linux/Redhat/Centos/Fedora repositories as standard.

Install the software as shown below on **all nodes** in the router tier.
```
% sudo yum localinstall --nogpgcheck mysql-router-commercial-8.0.20-1.1.el7.x86_64.rpm
% sudo yum localinstall --nogpgcheck mysql-shell-commercial-8.0.20-1.1.el7.x86_64.rpm
% sudo yum install pcs pacemaker resource-agents
```
Corosync is installed as a dependency of Pacemaker. Additionally we install pcs (to manage the Pacemaker cluster) and resource-agents (which provide an interface to the systemd managed MySQL Router). The cluster will be stateless and so there is no need to install fencing agents.

For information: the install of the above software will see the creation of three accounts:
```diff
% tail -3 /etc/passwd
- tss:x:59:59:Account used by the trousers package to sandbox the tcsd daemon:/dev/null:/sbin/nologin
- hacluster:x:189:189:cluster user:/home/hacluster:/sbin/nologin
- mysqlrouter:x:995:991:MySQL Router:/var/lib/mysqlrouter:/bin/false
%
```

### Security Considerations: firewalld, selinux and environmental
In order for application servers and other clients to connect to a MySQL Router the following ports need to be opened:
 * 6446/tcp - SQL protocol for read-write connections
 * 6447/tcp - SQL protocol for read-only connections
 * 64460/tcp - X protocol (for XDevAPI Document Store users) for read-write connections (this also allows SQL sessions)
 * 64470/tcp - X protocol (for XDevAPI Document Store users) for read-only connections (this also allows SQL sessions).
 
Pacemaker needs to communicate between the nodes using a variety of ports for both TCP and UDP protocols. Fortunately, Pacemaker is a well known package and the Linux firewall can be opened appropriately if the high-availability service is specified. Using this service rather than specifying ports should also insulate as from any changes to ports that might come about as a result of an upgrade, etc. The list of ports that will be opened for Enterprise Linux 7 (i.e. RedHat, Centos, Oracle Linux, etc.) can be found here: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/high_availability_add-on_reference/s1-firewalls-haar). 

On each node of the MySQL Router tier run the following commands:
```diff
% sudo firewall-cmd --zone=public --add-port=6446/tcp --permanent
% sudo firewall-cmd --zone=public --add-port=6447/tcp --permanent
% sudo firewall-cmd --zone=public --add-port=64460/tcp --permanent
% sudo firewall-cmd --zone=public --add-port=64470/tcp --permanent
% sudo firewall-cmd --permanent --add-service=high-availability
% sudo systemctl restart firewalld
% sudo firewall-cmd --list-ports
- 6446/tcp 64460/tcp 6447/tcp 64470/tcp
% sudo firewall-cmd --list-services
- dhcpv6-client high-availability ssh
%
```
The last two commands confirm that the ports have been opened for MySQL Router and that the high-availability service is in place.

No change to the selinux mode (enforcing | permissive | disabled) is required because our cluster implementation is stateless (i.e. no need to write to storage).

If implementing in a Cloud Environment then it may be necessary to make changes to virtual networking firewalls. For example, in the Oracle Cloud a rule must be in place to allow TCP and UDP traffic to run across the virtual network being used.

**Naming Services**: the test environment uses DNS to resolve hostnames into IP addresses. If you don't have a naming service then you will have to enter the names and IP addresses of each router node and indeed each database node into /etc/hosts. Clients of the router tier will need to be made aware of the floating IP address.

### Bootstrapping of MySQL Router
The MySQL Router instances need to be bootstrap with respect to the backend database tier they will connect to. In this case a 3 node InnoDB Cluster is being used. It's Primary node is called ic1 and it's cluster administrator user account is clustadm@ic1. For each node in the MySQL Router cluster, run the following
```
% sudo mysqlrouter --user mysqlrouter --force --bootstrap clusteradm@ic1
% sudo systemctl restart mysqlrouter
```
**Test MySQL Router Instance Connectivity**:

To test MySQL Router connectivity with the backend database tier, the database tier needs to be primed with some data and a user account. On the **primary node** of the **InnoDB Cluster**:
 * create a document store schema and give it a collection
 * create a database and give it a table
 * create a user and grant that user select, insert, update and delete on both the schema and database. Note the user should be able to log in from anywhere.

For example:
```diff
% hostname
- ic1
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
+ Using MySQL shell as a local client connect to the database using each of the MySQL Router ports: 6446 (SQL RW), 6447 (SQL RO), 64460 (X protocol RW), 66470 (X protocol RO)
% hostname
- rt1
% mysqlsh --uri stuart@localhost:6446         # Change port to 6447, 64460 and 64470 as required
- Please provide the password for 'stuart@localhost:6446': *********
  MySQL  localhost:6446 ssl  JS >

+ Test 1: Toggle to SQL and check which node of the InnoDB Cluster we are connected to. 
+         For connections on ports 6446 and 64460 we should be on the Primary node (RW)
+         For connections on ports 6447 and 64470 we should be on one of the Secondary nodes (RO)
 MySQL  localhost:6446 ssl  JS > \sql
- Switching to SQL mode...
 MySQL  localhost:6446 ssl  SQL > select @@hostname;
- +------------+
- | @@hostname |
- +------------+
- | ic1        |
- +------------+
- 1 row in set (0.0002 sec)
 MySQL  localhost:6446 ssl  SQL >
 
+ Test 2: Toggle to JavaScript and check whether you can connect to Document Store.
+         Only connections on the X Protocol ports 64460 and 64470 should be allowed connections
+         Connections on classic SQL ports (6446 and 6447) will receive error messages
 MySQL  localhost:6446 ssl  SQL > \js
- Switching to JavaScript mode...
 MySQL  localhost:6446 ssl  JS > var schema = session.getSchema("ancestors")
- Invalid object member getSchema (AttributeError)
 MySQL  localhost:6446 ssl  JS >
 
+ Test 3: If the schema object was obtained in test 2, access its collection and then add a document to it.
+         This cannot be done for connections on classic SQL ports (6446 and 6447) given they won't have been able to create the schema object.
+         This test will work in its entirety for connections on 64460 because they are read-write. 
+         Connections on port 64470 will only be able to do the query part (i.e. the find()) because connections on this port are read-only
 MySQL  localhost:64460+ ssl  JS > var collection = schema.getCollection("flintstones")
 MySQL  localhost:64460+ ssl  JS > collection.add({"name": "Fred", "type": "Early Human"})
- Query OK, 1 item affected (0.0062 sec)
 MySQL  localhost:64460+ ssl  JS > collection.find()
- {
-     "_id": "00005ef4b9130000000000000001",
-     "name": "Fred",
-     "type": "Early Human"
- }
- 1 document in set (0.0005 sec)
 MySQL  localhost:64460+ ssl  JS >

+ Test 4: Toggle to SQL mode and use the routertest database
+         This will work for all port types
 MySQL  localhost:6446 ssl  JS > \sql
 MySQL  localhost:6446 ssl  SQL > use routertest;
- Default schema set to `routertest`.
- Fetching table and column names from `routertest` for auto-completion... Press ^C to stop.
 MySQL  localhost:6446 ssl  routertest  SQL >
 
+ Test 5: Insert a row to routertest's table, then query it
+         This test will work in its entirety for connections using ports 6446 and 64460 because they are read-write
+         Connections using ports 6447 and 64470 will not be able to do the insert but will be able to do the query because they are read-only
 MySQL  localhost:6446 ssl  routertest  SQL > insert into t1 (name) values ("Stuart");
- Query OK, 1 row affected (0.0058 sec)
 MySQL  localhost:6446 ssl  routertest  SQL > select * from t1;
- +----+--------+
- | id | name   |
- +----+--------+
- |  1 | stuart |
- +----+--------+
- 1 row in set (0.0005 sec)
 MySQL  localhost:6446 ssl  routertest  SQL > \q

+ Now repeat the above until all MySQL Router nodes and ports have been tested.
```

### Setup and Configuration of the Cluster

**Initial Setup of the Pacemaker Cluster**:

Set the password for the hacluster user account **on each of the MySQL Router nodes**:
```diff
% sudo passwd hacluster
 New password: 
 Retype new password:
- passwd: all authentication tokens updated successfully.
```
In the example above the password used was MyPa55wd! - this will be used by Pacemaker to authenticate the nodes to the cluster and each other (see below).

Now start the pcsd service **on each MySQL Router node** in order that we can create the cluster. Assuming the start is successful then enable the service so that it automatically restarts on a reboot. 
```
% sudo systemctl start pcsd.service
% sudo systemctl enable pcsd.service
```

With the pcsd service now running on each node, the cluster can be created. To begin, set up authentication between the nodes using the hacluster user, then create the cluster and finally start it. On **one node of the cluster**:
```diff
% sudo pcs cluster auth rt1 rt2 rt3 -u hacluster -p MyPa55wd!
- rt1: Authorized
- rt2: Authorized
- rt3: Authorized
% sudo pcs cluster setup --name mysqlroutercluster rt1 rt2 rt3    
- Destroying cluster on nodes: mrt1, mrt2...
- rt1: Stopping Cluster (pacemaker)...
- rt2: Stopping Cluster (pacemaker)...
- rt3: Stopping Cluster (pacemaker)...
- rt1: Successfully destroyed cluster
- rt2: Successfully destroyed cluster
- rt3: Successfully destroyed cluster

- Sending 'pacemaker_remote authkey' to 'mrt1', 'mrt2'
- rt1: successful distribution of the file 'pacemaker_remote authkey'
- rt2: successful distribution of the file 'pacemaker_remote authkey'
- rt3: successful distribution of the file 'pacemaker_remote authkey'
- Sending cluster config files to the nodes...
- rt1: Succeeded
- rt2: Succeeded
- rt3: Succeeded

- Synchronizing pcsd certificates on nodes mrt1, mrt2...
- rt1: Success
- rt2: Success
- rt3: Success
- Restarting pcsd on the nodes in order to reload the certificates...
- rt1: Success
- rt2: Success
- rt3: Success

% sudo pcs cluster start --all
- rt1: Starting Cluster (corosync)...
- rt2: Starting Cluster (corosync)...
- rt3: Starting Cluster (corosync)...
- rt1: Starting Cluster (pacemaker)...
- rt2: Starting Cluster (pacemaker)...
- rt3: Starting Cluster (pacemaker)...
%
```
Give the cluster ~30 seconds to establish itself and then check its status. All the nodes must be online. 
```diff
% sudo pcs status
- Cluster name: mysqlroutercluster

- WARNINGS:
- No stonith devices and stonith-enabled is not false

- Stack: corosync
- Current DC: rt1 (version 1.1.21-4.el7-f14e36fd43) - partition with quorum
- Last updated: Thu Jul  2 13:51:38 2020
- Last change: Thu Jul  2 13:51:38 2020 by hacluster via crmd on rt1

- 3 nodes configured
- 0 resources configured

- Online: [ rt1 rt2 rt3 ]

- No resources


- Daemon Status:
-  corosync: active/disabled
-  pacemaker: active/disabled
-  pcsd: active/enabled
%
```
Some points to note:
* The cluster can be monitored from any node using either the status command as shown or **crm_mon**. If crm_mon is run then it will act as a console and continue to be updated. An alternative usage is to specify the number of times you want it to sample before exiting. For example to take a snapshot you would specify "one" as follows: crm_mon -1
* From the above status output we can see that 
  * We have warnings with respect to stonith (Shoot The Other Node In The Head - used for split brain) and these will need to be addressed. 
  * Quorum is in place, however, for our cluster we need to change the default behaviour when quorum is lost. 
  * We have a cluster but no resources are configured. Configuring resources (i.e. providing a VIP, associating the MySQL Router service with the cluster) will be done as one of the next steps.
  * We can also see that both pacemaker and corosync daemons are active/disabled. All this means is that these daemons are running (under systemd) but they have not been enabled to allow systemd to restart them upon reboot, etc.

**Configure Pacemaker Properties**:

The following properties were configured (see after the code block for the reasoning behind these settings)
```
% sudo pcs property set no-quorum-policy=ignore
% sudo pcs property set stonith-enabled=false
% sudo pcs resource defaults migration-threshold=1
```

The no-quorum-policy default value is stop, meaning that all resources (e.g. mysqlrouter) in the cluster will be stopped if the cluster does not have quorum. For what is an effectively stateless service this seems overly aggressive: why would you pull down a perfectly good database connection just because the cluster cannot achieve quorum? Available options include:
* ignore    continue all resource management	
* freeze    continue all resource management but do not recover resources from nodes not in the affected partition
* stop      stop all resources in the affected cluster partition
* suicide   fence all nodes in the affected cluster partition

For our purposes (single, simple, stateless application) ignore would seem to be the best fit.

When a resource is created it can be configured so that it will move to a new node after a defined number of failures by setting the migration-threshold option for that resource. Once the threshold has been reached, this node will no longer be allowed to run the failed resource until:
* The administrator manually resets the resource's failcount using the pcs resource failcount command.
* The resource's failure-timeout value is reached.
The value of migration-threshold is set to INFINITY by default. INFINITY is defined internally as a very large but finite number. A value of 0 disables the migration-threshold feature.

For our purposes we want to failover (so 0 is a wrong value) and we want the failover decision to be made on the first failure of MySQL Router and so a value of 1 is being used.

Stop failures are slightly different and crucial. If a resource fails to stop and STONITH is enabled, then the cluster will fence the node in order to be able to start the resource elsewhere. If STONITH is not enabled, then the cluster has no way to continue and will not try to start the resource elsewhere, but will try to stop it again after the failure timeout.

For the MySQL Router Tier there is no session state and so there is no need to worry about fencing and split brain. Therefore, STONITH can be disabled.

**Assigning Resources to the Pacemaker Cluster**:
Two resources will be added to the cluster: 
* a floating IP address called, Router_VIP
* the mysqlrouter application. 

With respect to the Router_VIP the following information is required:
* An IP address.
* The netmask for the subnet the IP address belongs; this needs to be in CIDR format.
* The name of the interface that the Router_VIP will be assigned to:
  * This is meant to be optional but in the environment being tested it seemed to be required (this may be a feature of a virtualized environment)
  * If the nic is specified, then it needs to be the same on each node in the cluster. 
  * To get the nic name run the linux command **ip addr** and select accordingly.
* An approrpriate interval in seconds. 

The mysqlrouter resource is managed through systemd. In the command to add this resource (see below) a **clone** parameter is used which indicates that MySQL Router will run on all nodes of the cluster.

```
% sudo pcs resource create Router_VIP ocf:heartbeat:IPaddr2 ip=10.0.0.101 cidr_netmask=16 nic=ens3 op monitor interval=5s
% sudo pcs resource create mysqlrouter systemd:mysqlrouter op monitor interval=5s clone
```
Once these resources have been added they will be colocated such that the floating IP must be present on the node running MySQL Router:
```
% sudo pcs constraint colocation add Router_VIP with mysqlrouter-clone score=INFINITY
```
The **score=INFINITY** parameter and value indicates that the source_resource, Router_VIP, must run on the same node as the target_resource, mysqlrouter.

**Readying the Cluster for Testing**:

Make sure each node of the cluster will restart when booted. **On every node**:
```
% sudo systemctl enable pacemaker
% sudo systemctl enable corosync
```

In order to make the cluster ready all that is needed is to restart it. **On one node**:
```
% sudo pcs cluster stop --all
% sudo pcs cluster start --all
```

Note: if you are deploying in the Oracle Cloud you will need to do some additional work (see below, after the Testing section).

## Testing
The following tests were conducted to prove the worth of the cluster:
* Basic Ping and Failover Testing
* Basic MySQL Connectivity and Failover Testing
* Basic Application Server Testing
* More Advanced Failover Testing
  * Killing MySQL Router on active node, failing over and recovery
  * Crashing the active node, failing over and recovery

### Basic Ping and Failover Testing
The purpose of these tests is to prove that the MySQL Router machines can be connected to via the floating IP and that the floating IP correctly fails over.

On a machine that is not part of the cluster (e.g. the client in the topology shown above) ping the floating IP address.
```diff
% hostname
- client
% ping 10.0.0.101
- PING 10.0.0.101 (10.0.0.101) 56(84) bytes of data.
- 64 bytes from 10.0.0.101: icmp_seq=1 ttl=64 time=0.280 ms
- 64 bytes from 10.0.0.101: icmp_seq=2 ttl=64 time=0.090 ms
- 64 bytes from 10.0.0.101: icmp_seq=3 ttl=64 time=0.106 ms
```

Log into one of the nodes in the cluster and set up continuous monitoring
```diff
% hostname
- rt2
% sudo crm_mon
- Stack: corosync
- Current DC: rt1 (version 1.1.21-4.el7-f14e36fd43) - partition with quorum
- Last updated: Tue Jul  7 11:01:16 2020
- Last change: Tue Jul  7 09:11:47 2020 by root via crm_resource on rt2

- 3 nodes configured
- 4 resources configured

- Online: [ rt1 rt2 rt3 ]

- Active resources:

- Router_VIP      (ocf::heartbeat:IPaddr2):       Started rt1
-  Clone Set: mysqlrouter-clone [mysqlrouter]
-      Started: [ rt1 rt2 rt3 ]
```

Log into one of the router nodes and move the Router_VIP from one node to the next: 

```diff
% hostname
- rt1
% sudo pcs resource move Router_VIP rt2
+ now move to another node
% sudo pcs resource move Router_VIP rt1
+ and another
% sudo pcs resource move Router_VIP rt3
```

The failover time will be approximately 5s. Watch the crm_mon output report the new location of the Router_VIP (i.e. Started rt1 will change to Started rt2 assuming the move to rt2 has been requested). Also watch the ping: it will pause whilst the failover is in place and then resume.

### Basic MySQL Connectivity and Failover Tests
The purpose of these tests is to prove connectivity to the MySQL Router application via the floating IP and that requests continue to be serviced after a failover event.

On the client machine run mysqlsh and perform a similar set of tests to those used when the router installs were being tested (see above). Note that the connection URL uses the MySQL account created earlier and the floating IP as an address.  

```diff
% hostname
- client
% mysqlsh --uri stuart@10.0.0.101:6446         # Change port to 6447, 64460 and 64470 as required
- Please provide the password for 'stuart@10.0.0.101:6446': *********
  MySQL  10.0.0.101:6446 ssl  JS >

+ Test 1: Toggle to SQL and check which node of the InnoDB Cluster we are connected to. 
+         For connections on ports 6446 and 64460 we should be on the Primary node (RW)
+         For connections on ports 6447 and 64470 we should be on one of the Secondary nodes (RO)
 MySQL  10.0.0.101:6446 ssl  JS > \sql
- Switching to SQL mode...
 MySQL  10.0.0.101:6446 ssl  SQL > select @@hostname;
- +------------+
- | @@hostname |
- +------------+
- | ic1        |
- +------------+
- 1 row in set (0.0002 sec)
 MySQL  10.0.0.101:6446 ssl  SQL >
 
+ Test 2: Toggle to JavaScript and check whether you can connect to Document Store.
+         Only connections on the X Protocol ports 64460 and 64470 should be allowed connections
+         Connections on classic SQL ports (6446 and 6447) will receive error messages
 MySQL  10.0.0.101:6446 ssl  SQL > \js
- Switching to JavaScript mode...
 MySQL  10.0.0.101:6446 ssl  JS > var schema = session.getSchema("ancestors")
- Invalid object member getSchema (AttributeError)
 MySQL  10.0.0.101:6446 ssl  JS >
 
+ Test 3: If the schema object was obtained in test 2, access its collection and then add a document to it.
+         This cannot be done for connections on classic SQL ports (6446 and 6447) given they won't have been able to create the schema object.
+         This test will work in its entirety for connections on 64460 because they are read-write. 
+         Connections on port 64470 will only be able to do the query part (i.e. the find()) because connections on this port are read-only
 MySQL  10.0.0.101:64460+ ssl  JS > var collection = schema.getCollection("flintstones")
 MySQL  10.0.0.101:64460+ ssl  JS > collection.add({"name": "Barney", "type": "Early Human"})
- Query OK, 1 item affected (0.0062 sec)
 MySQL  10.0.0.101:64460+ ssl  JS > collection.find()
- {
-     "_id": "00005ef4b9130000000000000001",
-     "name": "Fred",
-     "type": "Early Human"
- }
- {
-    "_id": "00005ef4b9130000000000000003",
-    "name": "Barney",
-    "type": "Early Human"
- }
- 2 documents in set (0.0009 sec)
 MySQL  10.0.0.101:64460+ ssl  JS >

+ Test 4: Toggle to SQL mode and use the routertest database
+         This will work for all port types
 MySQL  10.0.0.101:6446 ssl  JS > \sql
 MySQL  10.0.0.101:6446 ssl  SQL > use routertest;
- Default schema set to `routertest`.
- Fetching table and column names from `routertest` for auto-completion... Press ^C to stop.
 MySQL  10.0.0.101:6446 ssl  routertest  SQL >
 
+ Test 5: Insert a row to routertest's table, then query it
+         This test will work in its entirety for connections using ports 6446 and 64460 because they are read-write
+         Connections using ports 6447 and 64470 will not be able to do the insert but will be able to do the query because they are read-only
 MySQL  10.0.0.101:6446 ssl  routertest  SQL > insert into t1 (name) values ("William");
- Query OK, 1 row affected (0.0058 sec)
 MySQL  10.0.0.101:6446 ssl  routertest  SQL > select * from t1;
- +----+---------+
- | id | name    |
- +----+---------+
- |  1 | Stuart  |
- |  2 | William |
- +----+---------+
- 2 rows in set (0.0005 sec)
 MySQL  10.0.0.101:6446 ssl  routertest  SQL > \q

+ Now repeat the above using the remaining ports, e.g. mysqlsh --uri stuart@10.0.0.101:6447
```

The above tests demonstrate that a client can connect to the clustered MySQL Router tier which in turn connects to the clustered MySQL Database tier.

Now do some simple failover tests.

Setup monitoring on one router node (i.e. use crm_mon as before). 

On the client log into MySQL database tier via the router tier. Toggle to SQL mode and run queries against the routertest database:

```diff
% hostname
- client
% mysqlsh --uri stuart@10.0.0.101:6446      
- Please provide the password for 'stuart@10.0.0.101:6446': *********
 MySQL  10.0.0.101:6446 ssl  JS > \sql
 MySQL  10.0.0.101:6446 ssl  SQL > use routertest; 
 MySQL  10.0.0.101:6446 ssl  routertest  SQL > select * from t1 limit 2;
- +----+---------+
- | id | name    |
- +----+---------+
- |  1 | Stuart  |
- |  2 | William |
- +----+---------+
- 2 rows in set (0.0005 sec)
 MySQL  10.0.0.101:6446 ssl  routertest  SQL >
```

Log into a node on the router tier and failover the floating IP address (i.e. sudo pcs resource move Router_VIP rt1 or rt2 or rt3). Continue to run the above query in the MySQL shell session. You should see output like this after a failover of the floating IP address:

```diff
 MySQL  10.0.0.101:64460+ ssl  routertest  SQL > select * from t1 limit 2;
- +----+---------+
- | id | name    |
- +----+---------+
- |  1 | Stuart  |
- |  2 | William |
- +----+---------+
- 2 rows in set (0.0008 sec)
 MySQL  10.0.0.101:64460+ ssl  routertest  SQL > select * from t1 limit 2;
- ERROR: 2006: MySQL server has gone away
- The global session got disconnected..
- Attempting to reconnect to 'mysqlx://stuart@10.0.0.101:64460'..
- The global session was successfully reconnected.
 MySQL  10.0.0.101:64460+ ssl  routertest  SQL > select * from t1 limit 2;
- +----+---------+
- | id | name    |
- +----+---------+
- |  1 | Stuart  |
- |  2 | William |
- +----+---------+
- 2 rows in set (0.0011 sec)
 MySQL  10.0.0.101:64460+ ssl  routertest  SQL > 
```

Notice how on failover of the IP the client loses connection but automatically reconnects allowing the query to be rerun. From this we can conclude that a Pacemaker Cluster does provide a highly available tier for MySQL Router. 

### Basic Application Server Testing

## Additional Work Required for the Oracle Cloud
**Problem statement**: when the active node fails over to a passive node, the floating IP address must be moved to this passive node in order for it to become the new active node. Pacemaker detects a failover event and makes changes to the new active node's IP stack to bring up the floating IP **but** Oracle virtual networking is reluctant to reassign the floating IP address to the new active node. This is understandable and in normal circumstances desirable because it prevents the same IP address being used on two or more interfaces in the same subnet. However, it's not helpful in the case of reassigning a floating IP.

**Solution**: explicitly instruct Oracle virtual networking to remove the floating IP address from the failed node and reassign it to the new active node. The Pacemaker stack install provides an Open Cluster Framework Resource Agent script, /usr/lib/ocf/resource.d/heartbeat/IPaddr2, whose purpose is to provide an interface to manage IP resources and we can use this to instruct the Oracle virtual network to behave as we want.

**Implementation**: install Oracle Cloud Infrastructure (OCI) Command Line Interface (CLI) utility on each node in order to provide a script interface to the Oracle Cloud so that when a failover event triggers the script file, /usr/lib/ocf/resource.d/heartbeat/IPaddr2, can make a call to OCI which will reassign the floating IP address.

To install and configure the OCI CLI follow the procedure detailed here: https://docs.cloud.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm. The OCL CLI must be installed on each node of the cluster

Before you can edit /usr/lib/ocf/resource.d/heartbeat/IPaddr2 you will need to obtain the OCID for the (virtual) nic which will host the floating IP. This can be found by logging into the OCI console and then navigating to VNIC Details (i.e. Compute > Instances > Instance Details > Attached VNICs > VNIC Details).

Once you have the VNIC's OCID insert the following lines of code into /usr/lib/ocf/resource.d/heartbeat/IPaddr2 immediately under the header comments (line 64 in the file):

```
##### OCI/IPaddr Integration
export LC_ALL=C.UTF-8
export LANG=C.UTF-8
/root/bin/oci network vnic assign-private-ip --unassign-if-already-assigned --vnic-id "ocid1.vnic.oc1.uk-london-1.abcd...va" --ip-address "10.0.0.101"
```
Some points to note:
* Do the above edit on each node of the cluster, keeping the floating IP address the same and changing the vnic-id to suit the host.
* Explicitly setting the locale variables to C.UTF-8 overcomes any issues with Python 3 (which the OCI CLI uses).
* The path to the OCI executable (shown as /root/bin/oci) may vary depending upon your install.

A quick scan of the web will find other versions of this OCI/Pacemaker solution. Typically the update to /usr/lib/ocf/resource.d/heartbeat/IPaddr2 they detail looks similar to the following rather than the 3 lines of code shown above:

```sh
##### OCI/IPaddr Integration
export LC_ALL=C.UTF-8
export LANG=C.UTF-8
SERVER=`hostname -s`
RT1_VNIC="ocid1.vnic.oc1.uk-london-1.abcd...va"
RT2_VNIC="ocid1.vnic.oc1.uk-london-1.abcd...nq"
RT3_VNIC="ocid1.vnic.oc1.uk-london-1.abcd...wa"
FLOATING_IP="10.0.0.101"
if [ $SERVER = "rt1" ]; then
        /root/bin/oci network vnic assign-private-ip --unassign-if-already-assigned --vnic-id $RT1_VNIC --ip-address $FLOATING_IP
elif [ $SERVER = "rt2" ]; then
        /root/bin/oci network vnic assign-private-ip --unassign-if-already-assigned --vnic-id $RT2_VNIC --ip-address $FLOATING_IP
else
        /root/bin/oci network vnic assign-private-ip --unassign-if-already-assigned --vnic-id $RT3_VNIC --ip-address $FLOATING_IP
fi
```
This version can be written once and then deployed unchanged to each node. Some may consider this consistent approach an advantage. However, if you follow the logic of the script you will see that a lot of redundant code is being added. That said, either approach is valid; it's up to you to decide which you use.

Once /usr/lib/ocf/resource.d/heartbeat/IPaddr2 has been update on each node, the cluster will be ready for use.
