# oss-redis
Playbook for OSS Redis Cluster.


Note this tutorial requires Redis version 3.0 or higher.
The PCF Redis tile is just single instance Redis. We need use the Redis cluster broker to import the existing Redis cluster into the PCF. And on this tutorial, we ony focus on how setup an OSS Redis Cluster.

The offical website is here: https://redis.io/

The following demo shows how to setup Redis Cluster on a Linux machine with docker and brief introduction about the Redis Cluter.

Please note: This demo only works for the Linux.

## Setup Redis Instance

Setup Docker by following doc:
https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-docker-ce-1

Create **Dockerfile**

```
FROM alpine:3.7
 
# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN addgroup -S redis && adduser -S -G redis redis
 
# grab su-exec for easy step-down from root
RUN apk add --no-cache 'su-exec>=0.2'
 
ENV REDIS_VERSION 4.0.9
ENV REDIS_DOWNLOAD_URL http://download.redis.io/releases/redis-4.0.9.tar.gz
ENV REDIS_DOWNLOAD_SHA df4f73bc318e2f9ffb2d169a922dec57ec7c73dd07bccf875695dbeecd5ec510
 
# for redis-sentinel see: http://redis.io/topics/sentinel
RUN set -ex; \
	\
	apk add --no-cache --virtual .build-deps \
		coreutils \
		gcc \
		linux-headers \
		make \
		musl-dev \
	; \
	\
	wget -O redis.tar.gz "$REDIS_DOWNLOAD_URL"; \
	echo "$REDIS_DOWNLOAD_SHA *redis.tar.gz" | sha256sum -c -; \
	mkdir -p /usr/src/redis; \
	tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1; \
	rm redis.tar.gz; \
	\
# disable Redis protected mode [1] as it is unnecessary in context of Docker
# (ports are not automatically exposed when running inside Docker, but rather explicitly by specifying -p / -P)
# [1]: https://github.com/antirez/redis/commit/edd4d555df57dc84265fdfb4ef59a4678832f6da
	grep -q '^#define CONFIG_DEFAULT_PROTECTED_MODE 1$' /usr/src/redis/src/server.h; \
	sed -ri 's!^(#define CONFIG_DEFAULT_PROTECTED_MODE) 1$!\1 0!' /usr/src/redis/src/server.h; \
	grep -q '^#define CONFIG_DEFAULT_PROTECTED_MODE 0$' /usr/src/redis/src/server.h; \
# for future reference, we modify this directly in the source instead of just supplying a default configuration flag because apparently "if you specify any argument to redis-server, [it assumes] you are going to specify everything"
# see also https://github.com/docker-library/redis/issues/4#issuecomment-50780840
# (more exactly, this makes sure the default behavior of "save on SIGTERM" stays functional by default)
	\
	make -C /usr/src/redis -j "$(nproc)"; \
	make -C /usr/src/redis install; \
	\
	rm -r /usr/src/redis; \
	\
	runDeps="$( \
		scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
			| tr ',' '\n' \
			| sort -u \
			| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
	)"; \
	apk add --virtual .redis-rundeps $runDeps; \
	apk del .build-deps; \
	\
	redis-server --version
 
RUN mkdir /data && chown redis:redis /data
VOLUME /data
WORKDIR /data
 
CMD ["redis-server"]

```

Build Redis instance images:

```docker build -f Dockerfile -t iceshi/redis:4.0.9 /home/ice/redis/rediscluster```

After build the image, note down the image ID.
```
Successfully built 3e40559cd5f7
Successfully tagged iceshi/redis:4.0.9
ice@ice-ubuntu:~/redis/rediscluster$ 
```

Create the redis.conf file:
```
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
```

In this demo, we have need use 6 instance to setup an cluster, so we have 6 folder for instance just contains the configure file.

Then we can start the instance:
```docker run -t -P --restart=always --net=host --privileged=true -v /home/ice/redis/rediscluster/7000/redis.conf:/etc/redis.conf --name redis7000  3e40559cd5f7 redis-server /etc/redis.conf```

After change to root, we can use the following command 
`netstat -tnlp | grep redis`

```
root@ice-ubuntu:/home/ice/redis/rediscluster/7005# netstat -tnlp | grep redis
tcp        0      0 0.0.0.0:7000            0.0.0.0:*               LISTEN      1923/redis-server
tcp        0      0 0.0.0.0:7001            0.0.0.0:*               LISTEN      5220/redis-server
tcp        0      0 0.0.0.0:7002            0.0.0.0:*               LISTEN      5667/redis-server
tcp        0      0 0.0.0.0:7003            0.0.0.0:*               LISTEN      5730/redis-server
tcp        0      0 0.0.0.0:7004            0.0.0.0:*               LISTEN      5804/redis-server
tcp        0      0 0.0.0.0:7005            0.0.0.0:*               LISTEN      5872/redis-server
tcp        0      0 0.0.0.0:17000           0.0.0.0:*               LISTEN      1923/redis-server
tcp        0      0 0.0.0.0:17001           0.0.0.0:*               LISTEN      5220/redis-server
tcp        0      0 0.0.0.0:17002           0.0.0.0:*               LISTEN      5667/redis-server
tcp        0      0 0.0.0.0:17003           0.0.0.0:*               LISTEN      5730/redis-server
tcp        0      0 0.0.0.0:17004           0.0.0.0:*               LISTEN      5804/redis-server
tcp        0      0 0.0.0.0:17005           0.0.0.0:*               LISTEN      5872/redis-server
```

## Install Ruby Env and setup cluster

Get ruby:
```
sudo apt-add-repository ppa:brightbox/ruby-ng
sudo apt-get update
sudo apt-get install ruby2.5

```


gem install redis


Using the following command in utils folder in source to create the Cluster:
```
./redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 \
127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
```

Result:
```
ice@ice-ubuntu:~/redis/rediscluster/redis-4.0.9/src$ ./redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 \
> 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
127.0.0.1:7000
127.0.0.1:7001
127.0.0.1:7002
Adding replica 127.0.0.1:7004 to 127.0.0.1:7000
Adding replica 127.0.0.1:7005 to 127.0.0.1:7001
Adding replica 127.0.0.1:7003 to 127.0.0.1:7002
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 29cf0907d1f7f8803daba72f037af88fac1c49a7 127.0.0.1:7000
   slots:0-5460 (5461 slots) master
M: 57f3e51a9f8777a202ec8b85ac0ac4b6d39d5b99 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
M: c607ccd9ba3c3ea4a0463f23a921df0f508a7205 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
S: 07668e2ee8b4f15b09a780cc87eab1862100bf59 127.0.0.1:7003
   replicates c607ccd9ba3c3ea4a0463f23a921df0f508a7205
S: 610460106eb5e2c4bfbb422f350a9b30f9cbd30b 127.0.0.1:7004
   replicates 29cf0907d1f7f8803daba72f037af88fac1c49a7
S: 62c30b82c2ef5e0325d56320607428187a7a51ec 127.0.0.1:7005
   replicates 57f3e51a9f8777a202ec8b85ac0ac4b6d39d5b99
Can I set the above configuration? (type 'yes' to accept): yes

>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join..
>>> Performing Cluster Check (using node 127.0.0.1:7000)
M: 29cf0907d1f7f8803daba72f037af88fac1c49a7 127.0.0.1:7000
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
M: c607ccd9ba3c3ea4a0463f23a921df0f508a7205 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
S: 610460106eb5e2c4bfbb422f350a9b30f9cbd30b 127.0.0.1:7004
   slots: (0 slots) slave
   replicates 29cf0907d1f7f8803daba72f037af88fac1c49a7
S: 62c30b82c2ef5e0325d56320607428187a7a51ec 127.0.0.1:7005
   slots: (0 slots) slave
   replicates 57f3e51a9f8777a202ec8b85ac0ac4b6d39d5b99
M: 57f3e51a9f8777a202ec8b85ac0ac4b6d39d5b99 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
S: 07668e2ee8b4f15b09a780cc87eab1862100bf59 127.0.0.1:7003
   slots: (0 slots) slave
   replicates c607ccd9ba3c3ea4a0463f23a921df0f508a7205
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```




After all instances be started, we can use the redis cli to manage the Cluster.



```sudo apt-get install redis-tools```

Then connect to Redis:
```redis-cli -p 7000```



Please check the "OSS Redis Brief Introduction" for more details.


