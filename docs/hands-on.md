# PostgreSQL High Availability and automatic failover using repmgr

In this blog post, I’ll show you how to set up a PostgreSQL high-availability cluster with automatic failover using Docker containers, a subnet network, and the repmgr tool. Our goal is to make it easy for people to understand and have a hands-on experience with PostgreSQL replica and failover.

The setup consists of **three servers(containers)**: **one primary server** and **two standby servers**. With the help of the **repmgr** tool, when the primary server fails, it will automatically elect a new primary server. This ensures that our PostgreSQL cluster remains highly available and capable of handling failover scenarios.

In the following sections, I’ll cover the basic concepts of PostgreSQL high availability, introduce the tools and technologies used in our setup, and guide you through the steps required to create your own PostgreSQL high-availability cluster with automatic failover. Let's get started!

*While the steps provided are useful for hands-on experience with PostgreSQL replication and failover, it's important to note that these configurations may not be suitable for a production environment. The configurations provided are more permissive to make the process simpler and easier to understand.*

## In simple words

1. It sets up a PostgreSQL high-availability cluster with automatic failover using three servers: one primary and two standby servers.
2. The repmgr tool is used to facilitate replica and failover.
3. The primary server is registered to the cluster, and the standby server is created by cloning the primary server.
4. Both servers are registered to the cluster using repmgr.
5. repmgrd monitors the cluster and facilitate automatic failover. It constantly checks the health of the primary server and the standby servers to ensure they are up and running. If the primary server goes down, repmgrd will automatically elect a new primary server from the available standby servers.
6. When the primary server fails, a standby server is promoted to take its place automatically.
7. This ensures the PostgreSQL cluster remains available and operational during failover scenarios.

## Prepare the servers:

### Start the servers (containers)

```bash
# star the three servers
# pg1 -> primary server
# pg2 -> standby server[PostgreSQL High Availability and automatic failove 270e7aa64a5b42febcf64610e49ada2d.md](PostgreSQL%20High%20Availability%20and%20automatic%20failove%20270e7aa64a5b42febcf64610e49ada2d.md)
# pg3 -> standby server
docker compose up
```

### Install PostgreSQL, repmgr and a text editor

```bash
# attach to the primary server (pg1)
docker exec -it pg1 bash
# and install the prerequisites
apt update && apt install postgresql-13 postgresql-13-repmgr vim —yes

# repeat it for pg2
docker exec -it pg2 bash
apt update && apt install postgresql-13 postgresql-13-repmgr vim —yes

# repeat it for pg3
docker exec -it pg3 bash
apt update && apt install postgresql-13 postgresql-13-repmgr vim —yes
```

### Set up the primary server

We need to tell to the PostgreSQL server that we want to work with replicas and high availability. For that we update these entires:

- **`listen_address`**: Allows the PostgreSQL server to accept connections from any address, making it possible for the primary server and standby servers to communicate with each other.
- **`max_wal_senders`** and **`max_replication_slots`**: These parameters control the number of connections to the primary server that can be made by standby servers to receive WAL data. They ensure that the standby servers are not overwhelmed with data and that the primary server can efficiently send WAL data to the standby servers.
- **`wal_level`** and **`hot_standby`**: These settings enable the primary server to write WAL data that is required for standby servers to work correctly. By setting **`hot_standby`** to on, the standby servers can be used for read-only queries, allowing for more efficient use of resources.
- **`archive_mode`** and **`archive_command`**: These settings are used to ensure that WAL files are archived and available for disaster recovery scenarios. **`archive_mode`** enables archiving of WAL files, while **`archive_command`** specifies how the archived files should be handled. In the provided configuration, **`archive_command`** is set to **`/bin/true`** to ensure that the archived files are not actually saved to disk, but are still available for disaster recovery purposes.

```bash
# use the postgres user to setup/start/stop
su postgres

# update /etc/postgresql/13/main/postgresql.conf with the following values:
listen_address = '*'
max_wal_senders = 10
max_replication_slots = 10
wal_level = 'hot_standby'
hot_standby = on
archive_mode = on
archive_command = '/bin/true'

# start the server
/etc/init.d/postgres start

createuser -s repmgr
createdb repmgr -O repmgr

# enable remote connection to the primary server
# editing /etc/postgresql/13/main/pg_hba.conf
# this will enable our subnet ip's to connect to the primary server
host    replication     repmgr          172.7.7.0/24            trust
host    repmgr          repmgr          172.7.7.0/24            trust
```

### Register the primary server into the cluster

With the PostgreSQL and repmgr configurations in place, we can now use the repmgr tool to register our primary server node into the cluster. This will enable repmgr to monitor the health of our primary node, as well as any standby nodes, and to facilitate automatic failover in the event of a primary node failure.

By running the repmgr primary register command on the primary server node, we will create a record of the primary node in the repmgr metadata, and allow it to be recognized by other nodes in the cluster. This will enable repmgr to perform tasks such as monitoring replication and tracking the status of the primary node.

```bash
# create a /etc/postgresql/13/main/repmgr.conf file with this content:
node_id=1
node_name=pg1
conninfo='host=172.7.7.11 user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/postgresql/data'
failover=automatic
promote_command='repmgr -f /etc/postgresql/13/main/repmgr.conf standby promote'
follow_command='repmgr -f /etc/postgresql/13/main/repmgr.conf standby follow'

# and register the node into the cluster
repmgr -f /etc/postgresql/13/main/repmgr.conf primary register

# output should be similar to:
INFO: connecting to primary database...
NOTICE: attempting to install extension "repmgr"
NOTICE: "repmgr" extension successfully installed
NOTICE: primary node record (ID: 1) registered

# check if the cluster is available via
repmgr -f /etc/postgresql/13/main/repmgr.conf cluster show

# output should be similar to:
ID | Name | Role    | Status    | Upstream | Location | Priority | Timeline | Connection string
----+------+---------+-----------+----------+----------+----------+----------+-------------------------------------------------------------
 1  | pg1  | primary | * running |          | default  | 100      | 1        | host=172.7.7.11 user=repmgr dbname=repmgr connect_timeout=2
```

### Set up the secondary server

With the primary server up and running and waiting for standby servers to join the cluster, the next step is to clone the primary server to create a standby server. This standby server will serve as a replica of the primary server, allowing it to operate in read-only mode and receive all changes made to the primary server.

Once the standby server is registered with the cluster, it will be able to synchronize data with the primary server and remain in a "hot standby" state, ready to take over as the primary server in the event of a failover. This will help ensure high availability and reliability for the PostgreSQL cluster, as well as provide increased read scalability by allowing multiple read replicas to be added to the cluster.

```bash
# we will use postgres user for all the next steps
su postgres

# extra step, make sure your standby server can connect to the primary server
psql 'host=172.7.7.11 user=repmgr dbname=repmgr connect_timeout=2'

# create a clone from the primary server to act as a standby server, 
# create a file repmgr.conf at /etc/postgresql/13/main with this content:
node_id=2
node_name=pg2
conninfo='host=172.7.7.12 user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/postgresql/data'
failover=automatic
promote_command='repmgr -f /etc/postgresql/13/main/repmgr.conf standby promote'
follow_command='repmgr -f /etc/postgresql/13/main/repmgr.conf standby follow'

# create a replica / clone from the primary server
cd /etc/postgresql/13/main
# dry run in case you want to test it before calling the clone
repmgr -h 172.7.7.11 -U repmgr -d repmgr -f repmgr.conf standby clone --dry-run

# clone
repmgr -h 172.7.7.11 -U repmgr -d repmgr -f repmgr.conf standby clone

# edit /etc/postgresql/13/main/postgresql.conf and
# set data_directory to the cloned location
data_directory = '/var/lib/postgresql/data'
# allow anyone to connect
listen_address = '*'

# start the standby server
/etc/init.d/postgresql start

# register the standby server (/etc/postgresql/13/main/repmgr.conf)
repmgr -f repmgr.conf standby register

# see the available cluster nodes (/etc/postgresql/13/main/repmgr.conf)
repmgr -f repmgr.conf cluster show
 ID | Name | Role    | Status    | Upstream | Location | Priority | Timeline | Connection string
----+------+---------+-----------+----------+----------+----------+----------+-------------------------------------------------------------
 1  | pg1  | primary | * running |          | default  | 100      | 1        | host=172.7.7.11 user=repmgr dbname=repmgr connect_timeout=2
 2  | pg2  | standby |   running | pg1      | default  | 100      | 1        | host=172.7.7.12 user=repmgr dbname=repmgr connect_timeout=2
```

### ************repeat the same steps above for the other standby server:************

```bash
# we will use postgres user for all the next steps
su postgres

# extra step, make sure your standby server can connect to the primary server
psql 'host=172.7.7.11 user=repmgr dbname=repmgr connect_timeout=2'

# create a clone from the primary server to act as a standby server, 
# create a file repmgr.conf at /etc/postgresql/13/main with this content:
node_id=3
node_name=pg3
conninfo='host=172.7.7.13 user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/postgresql/data'
failover=automatic
promote_command='repmgr -f /etc/postgresql/13/main/repmgr.conf standby promote'
follow_command='repmgr -f /etc/postgresql/13/main/repmgr.conf standby follow'

# create a replica / clone from the primary server
cd /etc/postgresql/13/main
# dry run in case you want to test it before calling the clone
repmgr -h 172.7.7.11 -U repmgr -d repmgr -f repmgr.conf standby clone --dry-run

# clone
repmgr -h 172.7.7.11 -U repmgr -d repmgr -f repmgr.conf standby clone

# edit /etc/postgresql/13/main/postgresql.conf and
# set data_directory to the cloned location
data_directory = '/var/lib/postgresql/data'
# allow anyone to connect
listen_address = '*'

# start the standby server
/etc/init.d/postgresql start

# register the standby server (/etc/postgresql/13/main/repmgr.conf)
repmgr -f repmgr.conf standby register

# see the available cluster nodes (/etc/postgresql/13/main/repmgr.conf)
repmgr -f repmgr.conf cluster show
ID | Name | Role    | Status               | Upstream | Location | Priority | Timeline | Connection string
----+------+---------+----------------------+----------+----------+----------+----------+-------------------------------------------------------------
 1  | pg1  | primary | * running            |          | default  | 100      | 1        | host=172.7.7.11 user=repmgr dbname=repmgr connect_timeout=2
 2  | pg2  | standby |   running            | pg1      | default  | 100      | 2        | host=172.7.7.12 user=repmgr dbname=repmgr connect_timeout=2
 3  | pg3  | standby |   running            | pg1      | default  | 100      | 1        | host=172.7.7.13 user=repmgr dbname=repmgr connect_timeout=2
```

### Enable repmgrd for monitoring and failover handling

To enable monitoring and automatic failover handling, we need to set up repmgrd on all the PostgreSQL nodes (pg1, pg2 and pg3).

Start by logging into each server and registering them with repmgrd using the following command:

```bash
repmgrd -f /etc/postgresql/13/main/repmgr.conf -d
```

### Final step

With everything up and running smoothly, it's time to simulate a primary server failure and see how the cluster recovers from the failure.

To do this, we can simulate a crash on the primary server by stopping the PostgreSQL service. Once the primary server goes down, repmgr will automatically promote the most suitable standby server as the new primary server, and the other standby servers will follow the new primary server. We can observe this by running the commando bellow on any of the standby servers, which will display the current status of the cluster.

```bash
repmgr -f /etc/postgresql/13/main/repmgr.conf cluster show
```
