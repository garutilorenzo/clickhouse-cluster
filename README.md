[![Clickhouse Cluster CI](https://github.com/garutilorenzo/clickhouse-cluster/actions/workflows/ci.yml/badge.svg)](https://github.com/garutilorenzo/clickhouse-cluster/actions/workflows/ci.yml)
[![GitHub issues](https://img.shields.io/github/issues/garutilorenzo/clickhouse-cluster)](https://github.com/garutilorenzo/clickhouse-cluster/issues)
![GitHub](https://img.shields.io/github/license/garutilorenzo/clickhouse-cluster)
[![GitHub forks](https://img.shields.io/github/forks/garutilorenzo/clickhouse-cluster)](https://github.com/garutilorenzo/clickhouse-cluster/network)
[![GitHub stars](https://img.shields.io/github/stars/garutilorenzo/clickhouse-cluster)](https://github.com/garutilorenzo/clickhouse-cluster/stargazers)

![Clickhouse Logo](https://garutilorenzo.github.io/images/clickhouse.png)

# ClickHouse cluster Docker environment

[ClickHouse®](https://clickhouse.tech/) is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP).

# How this repo work

This simple docker environment shows how clickhouse  cluster works.
The cluster is made by two clickhouse services (click-one and click-two) and one zookeeper service (needed for replicated tables).
All configurations are stored on config directory at the same level of this README. In this directory you can find two directories one for each clickhouse service.

# Cluster description

This setup is meaded by two cluster:

* replicated
* distributed

the cluster configuration is located in cluster.xml on both config directories:

```
<yandex>
   <default_replica_path replace="replace">/clickhouse/tables/{shard}/{database}/{table}</default_replica_path>
   <default_replica_name replace="replace">{replica}</default_replica_name>
   <remote_servers replace="replace">
        <replicated>
            <shard>
              <replica>
                  <host>click-one</host>
                  <port>9000</port>
               </replica>
               <replica>
                  <host>click-two</host>
                  <port>9000</port>
               </replica>
             </shard>
        </replicated>
        <distributed>
            <shard>
              <replica>
                  <host>click-one</host>
                  <port>9000</port>
               </replica>
            </shard>
            <shard>
               <replica>
                  <host>click-two</host>
                  <port>9000</port>
               </replica>>
             </shard>
        </distributed>
    </remote_servers>
</yandex>
```

[Replicated](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/replication/) cluster is made by 2 servers, both of them are in the same shard this means that each table will be replicate 1 to 1 form one server to another.
[Distributed](https://clickhouse.tech/docs/en/engines/table-engines/special/distributed/) cluster is also made by two servers, but the servers are spread in two shard. **NOTE:** This config is not safe for production environments! Each shard in this configuration has no replica, this means that if we lose a server the entire distributed cluster will be inaccessible and we can have a possible data loss.

# Start the cluster

Use this command to start the cluster:

```
docker-compose up -d
```

on click-one container the sql directory is mounted on /docker-entrypoint-initdb.d path. 
The files:

* distributed.sql - creates tutorial_distributed database and then tutorial_distributed.hits_v1 table
* replicated.sql - creates tutorial_replicated database and then tutorial_replicated.hits_v1 table

This table is the same used on clickhouse [tutorial](https://clickhouse.tech/docs/en/getting-started/tutorial/)
You can then download the sample dataset:

```
curl https://datasets.clickhouse.tech/hits/tsv/hits_v1.tsv.xz | unxz --threads=`nproc` > hits_v1.tsv
```

and inject data inside the cluster:

```
docker-compose exec click-one clickhouse-client --query "INSERT INTO tutorial_distributed.hits_v1 FORMAT TSV" --max_insert_block_size=100000 < hits_v1.tsv
docker-compose exec click-one clickhouse-client --query "INSERT INTO tutorial_replicated.hits_v1 FORMAT TSV" --max_insert_block_size=100000 < hits_v1.tsv
```

# Cleanup

Use this command to clean all data and stop the containers:

```console
docker-compose down -v
```

# Notes about docker image

In this setup i've used a custom docker image.
There are some difference between the official image:

* added healtcheck on Dockerfile
* added sticky bit on /var/lib/clickhouse /var/log/clickhouse-server /etc/clickhouse-server /etc/clickhouse-client directories
* entrypoint.sh refectoring