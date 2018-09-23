# oss-redis
Playbook for OSS Redis Cluster.


Note this tutorial requires Redis version 3.0 or higher.
The PCF Redis tile is just single instance Redis. We need use the Redis cluster broker to import the existing Redis cluster into the PCF. And on this tutorial, we ony focus on how setup an OSS Redis Cluster.

The offical website is here: https://redis.io/

The following demo shows how to setup Redis Cluster on a Linux machine with docker and brief introduction about the Redis Cluter.

Please note: for Mac limitation, I still working on how to do it.

Setup Docker by following doc:
https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-docker-ce-1

Build Redis instance images:

docker build -f Dockerfile -t iceshi/redis:4.0.9 /home/ice/redis/rediscluster

Dockerfile is in the same folder.

Create the redis.conf file:
port 7000 ## change port number for each instance
pidfile /var/run/redis_6379.pid
maxmemory 50m
appendonly yes
################################ REDIS CLUSTER  ###############################
cluster-enabled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 5000
# cluster-slave-validity-factor 10
# cluster-migration-barrier 1
# cluster-require-full-coverage yes
# cluster-slave-no-failover no
 
################################## SLOW LOG ###################################
slowlog-log-slower-than 10000
slowlog-max-len 128


In this demo, we have need use 6 instance to setup an cluster, so we have 6 folder for instance just contains the configure file.

Then we can start the instance:
docker run -t -P --restart=always --net=host --privileged=true -v /home/ice/redis/rediscluster/7000/redis.conf:/etc/redis.conf --name redis7000  3e40559cd5f7 redis-server /etc/redis.conf

After all instances be started, we can use the redis cli to manage the Cluster.

sudo apt-get install redis-tools

Then connect to Redis:
redis-cli -p 7000


