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

## How to Create a HA Tier of MySQL Routers on Linux
The following details the stack and the operations needed to create a HA Cluster of MySQL Routers, as well as the testing conducted.

### MySQL Router Tier Stack
A three node HA Cluster of MySQL Router nodes was created in the Oracle Cloud. Each node was an Oracle Compute Instance (i.e. a Virtual Machine) with the following stack:

  * Virtual machine specification: 1 core OCPU, 16 GB memory, 1 Gbps network bandwidth, nom 50GB storage
  * OS: Oracle Linux 7.8, kernel rev 4.14.35-1902.303.4.1.el7uek.x86_64
  * MySQL Router 8.0.20
  * MySQL Shell 8.0.20 (optional - used to test router connectivity)
  * Pacemaker 1.1.21
  
Notes: 
1. Oracle Linux is a variant of Red Hat Enterprise Linux and as such it is expected that implementing a HA MySQL Router tier on either Red Hat, Centos or Fedora operating systems will work.
2. The Ubuntu OS also has a Pacemaker implementation and so it is assumed that this will also work.
3. Oracle Cloud: a small amount of additional integration work was required in order for the solution to work with Oracle Cloud's virtual network. This is detailed at the end of this document. It is anticipated that **no additional work** would be required with physical servers. Depending on how virtual networking is implemented by other cloud vendors there may be some similar work required. 

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

For simplicity we will put selinux into permissive mode. Do this on **all nodes** of the router tier
```
% ### Check to see what selinux mode you are in
% getenforce
enforcing
% ### If getenforce returns disabled or permissive then you are good to go, 
% ### however, if it reports enforcing you will need to edit the SELINUX= setting to permissive.
% ### If you do not like vim or nano, then use sed as shown:
% sudo sed --in-place=.bak 's/^SELINUX=.*$/SELINUX=permissive/' /etc/selinux/config
% ### Check that the config file looks the same as that detailed below.
% cat /etc/selinux/config
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=permissive
# SELINUXTYPE= can take one of three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted

% ### If it does not then the sed command should have created a copy of the original - /etc/selinux/config.bak
% ### Copy this back to /etc/selinux/config and correct the error accordingly.
% ###
% ### Assuming there are no errors, reboot the node
% sudo reboot
```

### Testing

The test infrastructure is detailed in the diagram below:
![](../master/images/testTopology.png)

### Oracle Cloud Specifics

In order for the floating VIP to be correctly assigned and reassigned to the nodes it was necessary to add the following lines 
