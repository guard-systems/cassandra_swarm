# Cassandra Swarm
Dockerfile for creating a Cassandra cluster with Docker swarm, and be able to deploy it as one service.  
It uses the DNS in Docker swarm to find all the other nodes and automatically sets them as seeds, and also sets the broadcast address to be the overlay network's ip address.  

## Docker hub
The image is available at vegah/cassandra_swarm on DockerHub.

## How to use
You will need to defined the service in the docker-compose.yml file, like below.  One environment variable is needed, and that is the name of the service.  Below it uses a template to get the name, and this should always work.
It sets up the volume with the Task.Slot in the path, which works - but of course will fail if a cassandra node is restarted on a different docker node.  Volumes and persistence will need to be solved depending on use case and setup.
### Example docker-compose.yml
``` 
version: '3'
services:
  cassandraswarm:
    image: vegah/cassandra_swarm
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
        max_attempts: 3
        window: 120s
      resources:
        limits:
          memory: 1000mb      
    environment:
      SERVICENAME: '{{.Service.Name}}'
    networks: 
      cassandra-net:
    volumes:
      - cassandravolume:/var/lib/cassandra
volumes:
  cassandravolume:
    external:
      name: 'cassandra_volume_{{.Task.Slot}}'
networks:
  cassandra-net:
```
Then you should be able to run:  
```bash
$  docker stack deploy -c docker-compose.yml cassandraswarm
Creating network cassandraswarm_cassandra-net
Creating service cassandraswarm_cassandraswarm
$ docker container ls
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS              PORTS                                         NAMES
38e5e26dee3a        vegah/cassandra_swarm:latest   "/docker-entrypoin..."   36 seconds ago      Up 34 seconds       7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcp   cassandraswarm_cassandraswarm.2.6x1kmaj2bfdqcvlpanlgpb6uz
7bb969548d87        vegah/cassandra_swarm:latest   "/docker-entrypoin..."   36 seconds ago      Up 34 seconds       7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcp   cassandraswarm_cassandraswarm.1.j90i4prqke95dyyrr6irsglds
be69aaa3349a        vegah/cassandra_swarm:latest   "/docker-entrypoin..."   36 seconds ago      Up 33 seconds       7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcp   cassandraswarm_cassandraswarm.3.w8spo9tfiozbhfw39jwg26sn5
$ docker exec 38e nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address   Load       Tokens       Owns (effective)  Host ID                               Rack
UN  10.0.0.3  68.2 KiB   256          68.2%             c68ab9b9-c45d-4078-ae29-9f3b13550344  rack1
UN  10.0.0.4  68.22 KiB  256          65.8%             dafad622-0017-4112-97c2-a72595b7ef19  rack1
UN  10.0.0.5  69.93 KiB  256          65.9%             366bae6e-f7f0-4403-9eb3-06cf0b59d785  rack1
```
# How does it work?
The image is based on https://github.com/docker-library/cassandra and the only difference is that the docker-entrypoint.sh file is changed to contain these lines:
```bash
# Wait 15 seconds to be sure that all nodes in the swarm has started and are available in DNS
sleep 15s
ownip="$(hostname)"
export CASSANDRA_SEEDS="$(nslookup tasks.$SERVICENAME | grep 'Address'| grep -v '127.0.0.1' | cut -d: -f2 | awk '{ print $1}' | sort | xargs | sed 's/ /,/g')"
export CASSANDRA_BROADCAST_ADDRESS=$ownip;
echo $CASSANDRA_SEEDS
echo $CASSANDRA_BROADCAST_ADDRESS
```
Which sets the environment varibles by looking at the dns.  The sleep 15s is needed to wait for docker to start all instances, so that DNS will report them.  15 seconds seem to be enough, but I guess this could vary.  

To be able to do a nslookup, it had to be installed - so the Dockerfile is based on the cassandra:latest image and then we do
```
FROM cassandra
RUN apt-get update && apt-get install -y dnsutils && rm -rf /var/lib/apt/lists/*
ADD ./files/docker-entrypoint.sh /
RUN chmod +x /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh","cassandra","-f"]
```
As you can see, apt-get install -y dnsutils install nslookup so that entrypoint.sh can use that.
  
