# Auto-swarm-creation

This is an example on how to create, initialize and start some service on a Docker Swarm cluster.


```
#!/bin/bash
docker-machine ls

#create 1st manager nodes
docker-machine create -d virtualbox  --swarm-experimental manager1
eval $(docker-machine env manager1)
docker swarm init --advertise-addr $(docker-machine ip manager1)

export SWARM_MANAGER_JOIN_TOKEN=$(docker swarm join-token -q manager)
export SWARM_WORKER_JOIN_TOKEN=$(docker swarm join-token -q worker)
export SWARM_MANAGER_IP=$(docker-machine ip manager1)

#create 2nd and 3th manager nodes
for N in 2 3; do
docker-machine create -d virtualbox  --swarm-experimental manager$N
eval $(docker-machine env manager$N)
docker swarm join --token $SWARM_MANAGER_JOIN_TOKEN $SWARM_MANAGER_IP
done

#create 4 to 7 worker nodes
for N in `seq 1 4`; do
docker-machine create -d virtualbox  --swarm-experimental  worker$N
eval $(docker-machine env worker$N)
docker swarm join --token $SWARM_WORKER_JOIN_TOKEN $SWARM_MANAGER_IP
done

#create and assign labels
eval $(docker-machine env manager1)
docker node update --label-add service=web worker1
docker node update --label-add service=web worker2
docker node update --label-add service=stress worker3
docker node update --label-add service=stress worker4

#create the networks
eval $(docker-machine env manager1)
docker network create --attachable --driver overlay stress-net
docker network create --attachable --driver overlay stress-net-2
docker network create --attachable --driver overlay stress-net-3
docker network create --attachable --driver overlay stress-net-4


#create a test service
docker service create --constraint node.role==worker --constraint node.labels.service==web --name nginx --network stress-net --replicas 3  -p 80:80  nginx

#create a test stress service
docker service create --constraint node.role==worker --constraint node.labels.service==stress --network stress-net   --name stress  --replicas 3  dockersec/siege  -c 2 http://nginx

```
# Happy Testing
