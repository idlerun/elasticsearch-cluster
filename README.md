---
reponame: elasticsearch-cluster
layout: repo
page: https://idle.run/elasticsearch-cluster
title: ElasticSearch Cluster on Docker 1.12 Swarm
tags: ip-camera, youtube
date: 2016-11-19
---

## Machines

For this example configuration, set up 3 Ubuntu virtual machines running in AWS:

- `manager1`
- `worker1`
- `worker2`

### Security Group:

- Create a security group with port 22 (ssh) inbound
- Note down the created security group ID
- Edit the security group and add rules to allow [required open ports](https://docs.docker.com/engine/swarm/networking/) between Docker nodes
    - Source address for all rules is the created security group ID
    - TCP inbound: 2377, 4789, 7946
    - UDP inbound: 2377, 4789

## OS Setup

### Install Docker

Install Docker according to [install instructions](https://docs.docker.com/engine/installation/linux/ubuntulinux/)

```bash
sudo apt-key adv --keyserver hkp://ha.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
echo "deb https://apt.dockerproject.org/repo ubuntu-xenial main" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt-get update
sudo apt-get install -y docker-engine
```

### Set required params

ElasticSearch will fail to start if `max_map_count` is set too low

```bash
# set for current session
sysctl -w vm.max_map_count=262144

# set at boot
sed -i '/vm.max_map_count/d' /etc/sysctl.conf
echo "vm.max_map_count = 262144" >> /etc/sysctl.conf
```

## Docker Swarm

Create a cluster according to the new (v1.12) [instructions](https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/)

On `manager1`

```bash
docker swarm init --advertise-addr <MANAGER-IP>
```

Replace `<MANAGER-IP>` with reachable address. For nodes in the same VPC this can (and should) be the private IP listed in AWS. If testing across regions then this may need to be the AWS public IP.

`swarm init` output will include the swarm join command which must be run on `worker1` and `worker2`

```bash
docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.99.100:2377
```

### Check swarm status

On `manager1` run commands and verify output is sane

```bash
docker info
```

```bash
docker node ls
```


## Docker Network

[Docker networkng overlay-security-model](https://docs.docker.com/engine/userguide/networking/overlay-security-model/)

Create an overlay network with encryption enabled for secure communication
between nodes.

```bash
docker network create \
  --driver overlay \
  --subnet 10.0.9.0/24 \
  --opt encrypted \
  enc-net
```

## ElasticSearch service

On a Docker master node.

Create the docker service in `global` mode. This will run one instance on each available node

```bash
docker service create \
  --mode global \
  --name elasticsearch \
  --network enc-net \
  --publish 9200:9200 \
  elasticsearch bash -c 'ip addr && IP=$(ip addr | awk -F"[ /]*" "/inet .*\/24/{print \$3}") && \
      echo publish_host=$IP && \
      exec /docker-entrypoint.sh -Enetwork.bind_host=0.0.0.0 -Enetwork.publish_host=$IP -Ediscovery.zen.minimum_master_nodes=2 -Ediscovery.zen.ping.unicast.hosts=tasks.elasticsearch'
```

#### Notes:
- The argument `--publish 9200:9200` exposes port 9200 on each host.
    - This argument can (and should) be removed in non-test environments where elasticsearch will be used with linked services
    - Use with caution. ElasticSearch warns **"Never expose an unprotected node to the public internet"**
- The `publish_host` discovery is scripted above to find the `10.0.9.X` address of the node (in the encrypted overlay network). Auto-discovery settings don't work correctly here to figure out the right address
- `tasks.elasticsearch` is a special DNS address for the elasticsearch cluster which lists out each of the node addresses
- Docs for Zen Discovery are available [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-zen.html)
- Docs for ElasticSearch network options are available [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-network.html)

### Docker Service Status

#### Verify service status on master node

```bash
docker service ps elasticsearch
```

Example output:

```
ID                         NAME               IMAGE          NODE      DESIRED STATE  CURRENT STATE           ERROR
8dhouf08d48t38vh94aqfw7wu  elasticsearch      elasticsearch  manager1  Running        Running 34 seconds ago
2wcm231uzcrcmvd1se3sakhtk   \_ elasticsearch  elasticsearch  worker1   Running        Running 34 seconds ago
cjw8npstwi1zqythx1rhr75fj   \_ elasticsearch  elasticsearch  worker2   Running        Running 34 seconds ago
```

#### Check logs for cluster status on each Docker node

```bash
docker logs $(docker ps -aq) | grep o.e.c.s.ClusterService
```

```
[2016-11-18T18:44:05,813][INFO ][o.e.c.s.ClusterService   ] [epbWb8a] new_master {epbWb8a}{epbWb8a-RE2lbXcLn8bN1Q}{thuHs3dBQWKcldQDG4_ijg}{10.0.9.3}{10.0.9.3:9300}, added {k9P08mI}{k9P08mIURR6eJNjSFbJc4w}{SOjMEOu5QhWXgRq0nsRYkA}{10.0.9.5}{10.0.9.5:9300},{knSkfDC}{knSkfDCnRA2i0-caESm2JA}{XgSMoGUnRdeYAsxWn7QzmQ}{10.0.9.4}{10.0.9.4:9300},}, reason: zen-disco-elected-as-master ([2] nodes joined)[{k9P08mI}{k9P08mIURR6eJNjSFbJc4w}{SOjMEOu5QhWXgRq0nsRYkA}{10.0.9.5}{10.0.9.5:9300}, {knSkfDC}{knSkfDCnRA2i0-caESm2JA}{XgSMoGUnRdeYAsxWn7QzmQ}{10.0.9.4}{10.0.9.4:9300}]
```

```
[2016-11-18T18:44:05,832][INFO ][o.e.c.s.ClusterService   ] [knSkfDC] detected_master {epbWb8a}{epbWb8a-RE2lbXcLn8bN1Q}{thuHs3dBQWKcldQDG4_ijg}{10.0.9.3}{10.0.9.3:9300}, added {k9P08mI}{k9P08mIURR6eJNjSFbJc4w}{SOjMEOu5QhWXgRq0nsRYkA}{10.0.9.5}{10.0.9.5:9300},{epbWb8a}{epbWb8a-RE2lbXcLn8bN1Q}{thuHs3dBQWKcldQDG4_ijg}{10.0.9.3}{10.0.9.3:9300},}, reason: zen-disco-receive(from master [master {epbWb8a}{epbWb8a-RE2lbXcLn8bN1Q}{thuHs3dBQWKcldQDG4_ijg}{10.0.9.3}{10.0.9.3:9300} committed version [1]])
```

```
[2016-11-18T18:44:05,832][INFO ][o.e.c.s.ClusterService   ] [k9P08mI] detected_master {epbWb8a}{epbWb8a-RE2lbXcLn8bN1Q}{thuHs3dBQWKcldQDG4_ijg}{10.0.9.3}{10.0.9.3:9300}, added {knSkfDC}{knSkfDCnRA2i0-caESm2JA}{XgSMoGUnRdeYAsxWn7QzmQ}{10.0.9.4}{10.0.9.4:9300},{epbWb8a}{epbWb8a-RE2lbXcLn8bN1Q}{thuHs3dBQWKcldQDG4_ijg}{10.0.9.3}{10.0.9.3:9300},}, reason: zen-disco-receive(from master [master {epbWb8a}{epbWb8a-RE2lbXcLn8bN1Q}{thuHs3dBQWKcldQDG4_ijg}{10.0.9.3}{10.0.9.3:9300} committed version [1]])
```

Note that all 3 nodes have detected `epbWb8a` as the master


## ElasticSearch Cluster

### Initial Health

On any node run the following

```bash
curl 'localhost:9200/_cluster/health?pretty'
```

Ensure that the cluster status is green, and that number of nodes is as expected

```
{
  "cluster_name" : "elasticsearch",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
...
```

### Test an Index

#### Add an index with shard and replica settings

```bash
curl -XPUT 'localhost:9200/blogs?pretty' -d'
{
   "settings" : {
      "number_of_shards" : 10,
      "number_of_replicas" : 1
   }
}'
```

##### To modify replica config later

```bash
curl -XPUT 'localhost:9200/blogs/_settings?pretty' -d'
{
   "settings" : {
      "number_of_replicas" : 2
   }
}'
```

#### Add some data

```bash
curl -XPUT 'localhost:9200/blogs/entry/123?pretty' -d'
{
  "title": "My first blog entry",
  "text":  "Just trying this out...",
  "date":  "2016/11/18"
}'
curl -XPUT 'localhost:9200/blogs/entry/456?pretty' -d'
{
  "title": "My second blog entry",
  "text":  "Still trying this out...",
  "date":  "2016/11/19"
}'
```

#### Verify replication

Check cluster health again, this time specifically for the `blogs` index

```bash
curl 'localhost:9200/_cluster/health/blogs?pretty'
```

```
{
  "cluster_name" : "elasticsearch",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 10,
  "active_shards" : 20,
```

## Kibana service

Create a service for kibana linked to the elasticsearch service

```
docker service create \
  --mode global \
  --name kibana \
  --network enc-net \
  --publish 5601:5601 \
  kibana
```

Note that no `--link` is required (or possible) for services. Instead all services are exposed on their network with their service name. Because we used the name `elasticsearch` which `kibana` expects by default, no additional config options are required.

For further config options see the [docker hub: kibana](https://hub.docker.com/_/kibana/) page

### Connect to Kibana

Use `ssh` to forward port 5601 to one of your nodes

```bash
ssh -NL 5601:localhost:5601 manager1
```

Open `http://localhost:5601` in your web browser

When prompted for the default index enter `blogs`

Note that the Kibana searches default to only `last 15 minutes`. This is changed with the date picker in the top right corner.

