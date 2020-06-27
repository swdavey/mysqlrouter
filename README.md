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
