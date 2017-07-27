# Auto creating DOCKER SWARM clusters on virtualbox and running services

This small project focuses on creating docker nodes, initialize docker swarm, creating a sample web service and traffic generation. The examples adds advanced features like labels, attachable overlays, etc ... to make it a little more advanced.

The script uses the latest version of docker-machine.
See https://docs.docker.com/machine/install-machine/ for installation guidelines. 
Obtain the latest version at https://github.com/docker/machine/releases/


```
#!/bin/bash
docker-machine ls

#create 1st manager node
docker-machine create -d virtualbox  --swarm-experimental manager1
eval $(docker-machine env manager1)
docker swarm init --advertise-addr $(docker-machine ip manager1)

export SWARM_MANAGER_JOIN_TOKEN=$(docker swarm join-token -q manager)
export SWARM_WORKER_JOIN_TOKEN=$(docker swarm join-token -q worker)
export SWARM_MANAGER_IP=$(docker-machine ip manager1)

#create 2nd and 3th manager node
for N in 2 3; do
docker-machine create -d virtualbox  --swarm-experimental manager$N
eval $(docker-machine env manager$N)
docker swarm join --token $SWARM_MANAGER_JOIN_TOKEN $SWARM_MANAGER_IP:2377
done

#create 4 to 7 worker nodes
for N in `seq 1 4`; do
docker-machine create -d virtualbox  --swarm-experimental  worker$N
eval $(docker-machine env worker$N)
docker swarm join --token $SWARM_WORKER_JOIN_TOKEN $SWARM_MANAGER_IP:2377
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

#create a test service
docker service create --constraint node.role==worker --constraint node.labels.service==web --name nginx --network stress-net --detach=false --replicas 3  -p 80:80  nginx

#create a test stress service
docker service create --constraint node.role==worker --constraint node.labels.service==stress --name stress --network stress-net  --detach=false --replicas 3  dockersec/siege  -c 2 http://nginx

```

When finished, point your docker client to a SWARM master
```
eval $(docker-machine env manager1)
```

List the docker nodes and swarm status
```
docker node ls

ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
5kik5m9hdd1tadfklktozs4z6     worker4             Ready               Active              
6jhi6g2jm1wxr43voophkcfr0     worker1             Ready               Active              
gmb3hgtjnmfycindfsn3i83s9     manager3            Ready               Active              Reachable
ikha8tk61t2ojvyfycaxa8jcq *   manager1            Ready               Active              Leader
madeihv8i14uzjl1jiloq9s6c     worker2             Ready               Active              
nnnp6joms8usimxjt48xzrkj8     manager2            Ready               Active              Reachable
sudr22u396wbmj05cw4b7xhi5     worker3             Ready               Active              

```



