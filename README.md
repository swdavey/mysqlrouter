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

An alternative is to use standard clustering technologies and approaches. For example: software that manages cluster membership and failover; a floating IP address such that client software has a consistent point of attachment. The remainder of this document details how this can be achieved on Linux servers. 

## How to Create a HA Tier of MySQL Routers on Linux

### Test Setup
In order to represent a production deployment the follwing architectural topology was created in a single virtual network in the Oracle Cloud.


* Front End
  * Comprising a single virtual machine working as a client sending cURL requests to the application tier and receiving responses accordingly.
  * MySQL 8.0.20 was installed in order to test the HA implementation of MySQL Router without having to go via the application server. In production this would not be required.
* Application Tier
  * Comprising a single virtual machine that hosts a RESTful Java application running in a JBoss Wildfly applicaton server. The application accepts the cURL requests from the client and in order to service these requests connects to the database tier via the highly available router tier. 
* Router Tier
  * Comprising three virtual machines each running MySQL Router in a highly available cluster; effectively the system under test.
  * Virtual machine specification: 1 core OCPU, 16 GB memory, 1 Gbps network bandwidth, 50GB storage
  * OS: Oracle Linux 7.8, kernel rev 4.14.35-1902.303.4.1.el7uek.x86_64
  * MySQL Router 8.0.20
  * MySQL Shell 8.0.20 (used to test router connectivity - could be removed if required)
  * Pacemaker 
* Database Tier
  * Comprising three virtual machines 
  * OS: Oracle Linux 7.8, kernel rev 4.14.35-1902.303.4.1.el7uek.x86_64
  * MySQL Server 8.0.20
  * MySQL Shell 8.0.20 (required to administer MySQL InnoDB Cluster)

Notes: 
1. Oracle Linux is a variant of Red Hat Enterprise Linux and as such it is expected that implementing a HA MySQL Router tier on either Red Hat, Centos or Fedora operating systems will work.
2. The Ubuntu OS also has a Pacemaker implementation and so it is assumed that this will also work.
3. Oracle Cloud: a small amount of additional integration work was required in order for the solution to work with Oracle Cloud's virtual network. This is detailed at the end of this document. It is anticipated that no additional work would be required with physical servers. Depending on how virtual networking is implemented by other cloud vendors there may be some similar work required. 

### Software Install of the Router Tier
Either the Enterprise or Community editions of MySQL software can be used. In 
The RPM packages for MySQL are the commercial versions (enterprise edition) and were downloaded prior to install.

```

```


### Security: firewalld and selinux

### Oracle Cloud Specifics

In order for the floating VIP to be correctly assigned and reassigned to the nodes it was necessary to add the following lines 
