
# PostgreSQL High Availability and Automatic Failover using repmgr

This project sets up a PostgreSQL high-availability cluster with automatic failover using Docker containers, a subnet 
network, and the repmgr tool. Our goal is to make it easy for people to understand and have a hands-on experience with 
PostgreSQL replica and failover.

## Overview

The setup consists of three servers (containers): one primary server and two standby servers. With the help of the 
repmgr tool, when the primary server fails, it will automatically elect a new primary server. This ensures that our 
PostgreSQL cluster remains highly available and capable of handling failover scenarios.

_Disclaimer_

While the steps provided are useful for hands-on experience with PostgreSQL replication and failover, it's important to 
note that these configurations may not be suitable for a production environment. The configurations provided are more 
permissive to make the process simpler and easier to understand.
