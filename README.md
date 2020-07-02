# Highly Available Deployment of MySQL Router

## Introduction
The purpose of this readme is to document how to set up a highly available mid-tier of MySQL Router instances.

MySQL Router is typically used in conjunction with either MySQL InnoDB Cluster or MySQL Replica Sets. 
Applications connect to a MySQL Router instance and that instance then transparently routes communications 
between the application and the backend (InnoDB Cluster or Replica Set).
MySQL Router keeps track of the backend's topology (i.e. nodes joining and leaving as well as each node's role: primary or secondary). By doing so it becomes aware of backend node failures and when such failures do it occur it will failover connections to a surviving node. 

## Deployment Approaches
MySQL Router can be deployed either with the application server or on its own mid-tier.

The advantage of deploying MySQL Router with the application is there is no need to worry about high availability: if the hosting machine is up and running correctly then, assuming MySQL Router has been configured and bootstrapped correctly, it will almost certainly continue to run. The disadvantage of deploying MySQL Router on application servers is that you have to do it for each and every application server. This is not too much of a problem if you only have small numbers of application servers but becomes more onerous with large numbers (e.g. administration effort to deploy and (perhaps more critically) maintain). 

To overcome the issue of deploying MySQL Router on large numbers of application servers MySQL Router can be deployed on a separate mid-tier. This mid-tier needs to be made highly-available in order to prevent it representing a Single Point Of Failure (SPOF) which would largely defeat the point of making the backend database highly available. Currently (June 2020. MySQL 8.0.20 being the most recent release), MySQL Router has no built-in clustering function. Therefore to make a HA MySQL Router tier third party solutions are required.

One method documented by MySQL is to use DNS-SRV records in conjuction with software such as Consul and dnsmasq. In practice this is difficult to implement and given it requires changes to DNS configurations is likely to be resisted in production environments.

An alternative is to use standard clustering technologies and approaches. For example: software that manages cluster membership and failover; a floating IP address such that client software has a consistent point of attachment. The remainder of this document details how this can be achieved on Linux servers using Pacemaker (Linux clusterware).

## Test Environment
The diagram below details the test environment upon which this document is based. Given the purpose of this document is to discuss how to create a highly available MySQL Router tier only that element will be discussed in detail and the other components will only be mentioned in support of that aim.

### MySQL Router Tier Stack
A three node HA Cluster of MySQL Router nodes was created in the Oracle Cloud. Each node was an Oracle Compute Instance (i.e. a Virtual Machine) with the following stack:

  * Virtual machine specification: 1 core OCPU, 16 GB memory, 1 Gbps network bandwidth, nom 50GB storage
  * OS: Oracle Linux 7.8, kernel rev 4.14.35-1902.303.4.1.el7uek.x86_64
  * MySQL Router 8.0.20
  * MySQL Shell 8.0.20 (optional - used to test router connectivity)
  * Pacemaker 1.1.21
  
Notes: 
1. Oracle Linux is a variant of Red Hat Enterprise Linux and as such it is expected that implementing a HA MySQL Router tier on either Red Hat, Centos or Fedora operating systems will work if configured in the same manner.
2. The Ubuntu OS also has a Pacemaker implementation and so it is assumed that this will also work.
3. Oracle Cloud: a small amount of additional integration work was required in order for the solution to work with Oracle Cloud's virtual network. This is detailed at the end of this document. It is anticipated that **no additional work** would be required with physical servers. Depending on how virtual networking is implemented in other cloud there may be some similar work required. 

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

### Security: firewalld and selinux
In order for application servers and other clients to connect to a MySQL Router the following ports need to be opened:
 * 6446/tcp - SQL protocol for read-write connections
 * 6447/tcp - SQL protocol for read-only connections
 * 64460/tcp - X protocol (for XDevAPI Document Store users) for read-write connections (this also allows SQL sessions)
 * 64470/tcp - X protocol (for XDevAPI Document Store users) for read-only connections (this also allows SQL sessions).
 
Pacemaker needs to communicate between the nodes using a variety of ports for both TCP and UDP protocols. Fortunately, Pacemaker is a well known package and the Linux firewall can be opened appropriately if the high-availability service is specified. 

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

No change to the selinux mode (enforcing | permissive | disabled) is required because the cluster is stateless (no need for fencing and split brain is not an issue).

### Naming Services
The test environment uses DNS (part of the Oracle Cloud) to resolve hostnames into IP addresses. If you don't have a naming service then you will have to enter the names and IP addresses of each router node and indeed each database node into /etc/hosts. Clients of the router tier will need to be made aware of the floating IP address.

### Bootstrapping of MySQL Router
To bootstrap the cluster we need to be aware of the backend database cluster or replica set being used. In this case a 3 node InnoDB Cluster is being used. It's Primary node is called ic1 and it's cluster administrator user account is clustadm@ic1. For each node in the MySQL Router cluster, run the following
```
% sudo mysqlrouter --user mysqlrouter --force --bootstrap clusteradm@ic1
% sudo systemctl restart mysqlrouter
```
### Test MySQL Router Instance Connectivity
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
Using MySQL shell as a local client connect to the database using each of the MySQL Router ports: 6446 (SQL RW), 6447 (SQL RO), 64460 (X protocol RW), 66470 (X protocol RO)
% hostname
rt1
% mysqlsh --uri stuart@localhost:6446         # Change port to 6447, 64460 and 64470 as required
Please provide the password for 'stuart@localhost:6446': *********
 MySQL  localhost:6446 ssl  JS >

@@ Test 1: Toggle to SQL and check which node of the InnoDB Cluster we are connected to. 
@@         For connections on ports 6446 and 64460 we should be on the Primary node (RW)
@@         For connections on ports 6447 and 64470 we should be on one of the Secondary nodes (RO)

 MySQL  localhost:6446 ssl  SQL > select @@hostname;
+------------+
| @@hostname |
+------------+
| ic1        |
+------------+
1 row in set (0.0002 sec)
 MySQL  localhost:6446 ssl  SQL >
 
~ Test 2: Toggle to JavaScript and check whether you can connect to Document Store.
~         Only connections on the X Protocol ports 64460 and 64470 should be allowed connections
~         Connections on classic SQL ports (6446 and 6447) will receive error messages
 MySQL  localhost:6446 ssl  SQL > \js
Switching to JavaScript mode...
 MySQL  localhost:6446 ssl  JS > var schema = session.getSchema("ancestors")
Invalid object member getSchema (AttributeError)
 MySQL  localhost:6446 ssl  JS >
 
 Test 3: If the schema object was obtained in test 2, access its collection and then add a document to it.
         This cannot be done for connections on classic SQL ports (6446 and 6447) given they won't have been able to create the schema object.
         This test will work in its entirety for connections on 64460 because they are read-write. 
         Connections on port 64470 will only be able to do the query part (i.e. the find()) because connections on this port are read-only
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

Test 4: Toggle to SQL mode and use the routertest database
        This will work for all port types
 MySQL  localhost:6446 ssl  JS > \sql
 MySQL  localhost:6446 ssl  SQL > use routertest;
Default schema set to `routertest`.
Fetching table and column names from `routertest` for auto-completion... Press ^C to stop.
 MySQL  localhost:6446 ssl  routertest  SQL >
 
 Test 5: Insert a row to routertest's table, then query it
         This test will work in its entirety for connections using ports 6446 and 64460 because they are read-write
         Connections using ports 6447 and 64470 will not be able to do the insert but will be able to do the query because they are read-only
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

Now repeat the above until all MySQL Router nodes and ports have been tested.
```

### Setup and Configure the Pacemaker Cluster
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

With the pcsd service running on each node, we can create the cluster. Firstly, set up authentication between the nodes then create the cluster and finally start it
```
% sudo pcs cluster auth rt1 rt2 rt3 -u hacluster -p MyPa55wd! --force
% sudo pcs cluster setup --force --name mysqlroutercluster rt1 rt2 rt3
% sudo pcs cluster start --all
```
### Testing

The test infrastructure is detailed in the diagram below:
![](../master/images/testTopology.png)

### Oracle Cloud Specifics

In order for the floating VIP to be correctly assigned and reassigned to the nodes it was necessary to add the following lines 
