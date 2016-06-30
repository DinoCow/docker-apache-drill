# docker-apache-drill

[![](https://imagelayers.io/badge/smizy/apache-drill:1.6-alpine.svg)](https://imagelayers.io/?images=smizy/apache-drill:1.6-alpine 'Get your own badge on imagelayers.io')

apache-drill docker image based on java:8-jre-alpine

## run drill server (pseudo-distributed mode) 

run on bridge network in single docker host

```
# network
docker netwrok create vnet

# zookeeper
for i in 1 2 3; do docker run \
--name zookeeper-$i \
--net vnet \
-h zookeeper-$i.vnet \
-d smizy/zookeeper:3.4-alpine \
-server $i 3 \
;done 

# drill
docker run \
--name drillbit-1 \
--net vnet \
-h drillbit-1.vnet \
-p 8047:8047 \
-e DRILL_HEAP=512M \
-e DRILL_MAX_DIRECT_MEMEORY=1G \
-e DRILL_ZOOKEEPER_QUORUM=zookeeper-1.vnet:2181,zookeeper-2.vnet:2181,zookeeper-3.vnet:2181 \
-d smizy/apache-drill:1.7-alpine 

# drill client
docker exec -it drillbit-1 drill-conf
0: jdbc:drill:> show databases;
+---------------------+
|     SCHEMA_NAME     |
+---------------------+
| INFORMATION_SCHEMA  |
| cp.default          |
| dfs.default         |
| dfs.root            |
| dfs.tmp             |
| sys                 |
+---------------------+
6 rows selected (2.349 seconds)
0: jdbc:drill:> !quit
```

# run drill server (distributed mode)

run on overlay network in multiple docker host with swarm
```

# create manager node
for i in 1 2 3; do \
  docker-machine create \
  -d virtualbox \
  --virtualbox-memory 384 \
  --engine-opt="cluster-store=consul://localhost:8500" \
  --engine-opt="cluster-advertise=eth1:2376" \
  --swarm \
  --swarm-discovery consul://localhost:8500 \
  --swarm-master \
  --swarm-opt replication \
  manager-$i 
  
  # consul server 
  docker $(docker-machine config manager-$i) run -d \
  --name consul-server \
  --net host \
  --restart unless-stopped \
  gliderlabs/consul-server:0.6 \
  -server -bootstrap-expect 3  \
  -bind $(docker-machine ip manager-$i) 
done

# consul-server join
docker $(docker-machine config manager-1) exec -it consul-server consul join \
  $(docker-machine ip manager-1) $(docker-machine ip manager-2) $(docker-machine ip manager-3)

# check consul members manager-*
docker $(docker-machine config manager-1) exec -it consul-server consul members

# create data node
for i in 1 2 3; do \
  docker-machine create \
  -d virtualbox \
  --virtualbox-memory 512 \
  --engine-opt="cluster-store=consul://localhost:8500" \
  --engine-opt="cluster-advertise=eth1:2376" \
  --swarm \
  --swarm-discovery consul://localhost:8500 \
  node-d-$i; 
  
  docker $(docker-machine config node-d-$i) run -d \
  --name consul-agent \
  --net host \
  --restart unless-stopped \
  gliderlabs/consul-agent:0.6 \
  -bind $(docker-machine ip node-d-$i)
  
  # consul-agent join
  docker $(docker-machine config node-d-$i) exec -it consul-agent consul join \
  $(docker-machine ip manager-1) $(docker-machine ip manager-2) $(docker-machine ip manager-3)    
done

# check consul members contain manager-* and node-d-*
docker $(docker-machine config node-d-1) exec -it consul-agent consul members
 
# registrator
for i in manager-1 manager-2 manager-3 for i in node-d-1 node-d-2 node-d-3; do \
  docker $(docker-machine config ${i}) run -d \
  --name registrator \
  --net host \
  --restart unless-stopped \
  -v /var/run/docker.sock:/tmp/docker.sock \
  gliderlabs/registrator -internal  consul://localhost:8500 
done 

# create overlay network
docker $(docker-machine config manager-1) network create -d overlay vnet

# zookeeper
for i in 1 2 3; do \
docker $(docker-machine config manager-$i) run \
--name zookeeper-$i \
--net vnet \
-h zookeeper-$i.vnet \
-e SERVICE_NAME=zookeeper \
-d smizy/zookeeper:3.4-alpine \
-server $i 3 
done

# journalnode
for i in 1 2 3; do \
docker $(docker-machine config manager-$i) run \
--name journalnode-$i \
--net vnet \
-h journalnode-$i.vnet \
--expose 8480 \
--expose 8485 \
-e SERVICE_NAME=journalnode \
-d smizy/hadoop-base:2.7.2-alpine \
entrypoint.sh journalnode 
done 

# namenode
for i in 1 2; do \
docker $(docker-machine config manager-$i) run \
--name namenode-$i \
--net vnet \
-h namenode-$i.vnet \
--expose 8020 \
--expose 50070 \
-e SERVICE_NAME=namenode \
-d smizy/hadoop-base:2.7.2-alpine \
entrypoint.sh namenode-$i 
done 

# datanode
for i in 1; do \
docker $(docker-machine config node-d-$i) run \
--name datanode-$i \
--net vnet \
-h datanode-$i.vnet \
--expose 50010 \
--expose 50020 \
--expose 50075 \
-e SERVICE_NAME=datanode \
-d smizy/hadoop-base:2.7.2-alpine \
entrypoint.sh datanode 
done 

# check hdfs sample program
docker exec -it -u hdfs datanode-1 hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.2.jar pi 3 3

# drill
for i in 1; do \
docker $(docker-machine config node-d-$i) run \
--name drillbit-$i \
--net vnet \
-h drillbit-$i.vnet \
-p 8047:8047 \
-e DRILL_HEAP=512M \
-e DRILL_MAX_DIRECT_MEMEORY=1G \
-e DRILL_ZOOKEEPER_QUORUM=zookeeper-1.vnet:2181,zookeeper-2.vnet:2181,zookeeper-3.vnet:2181 \
-e SERVICE_NAME=drill \
-d smizy/apache-drill:1.6-alpine 
done

# check drill webui
open http://$(docker-machine ip node-d-1):8047

# Query json data on hdfs with apache-drill
$ echo '{ a:1, b:2, c:3}' > test.json
$ docker exec -it -u hdfs datanode-1 bash
bash-4.3$ hdfs dfs -mkdir /user/hdfs
bash-4.3$ hdfs dfs -mkdir /user/hdfs/output
bash-4.3$ hdfs dfs -put test.json /user/hdfs/output/

# update dfs storage setting (on drill webui)

{
  "type": "file",
  "enabled": true,
  "connection": "hdfs://namenode-1.vnet:8020/",
  "config": null,
  "workspaces": {
    "root": {
      "location": "/user/hdfs",
      "writable": false,
      "defaultInputFormat": null
    },
    "tmp": {
      "location": "/tmp",
      "writable": true,
      "defaultInputFormat": null
    }
  },
  :
  :
  
## run query from WEB UI
select * from dfs.root.`output/test.json`

```

# mustache.sh LICENSE
* BSD License. See LICENSE.mustache.
* Source: https://github.com/rcrowley/mustache.sh
* Copyright 2011 Richard Crowley. All rights reserved.