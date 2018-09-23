# oss-redis
Playbook for OSS Redis Cluster.


Note this tutorial requires Redis version 3.0 or higher.
The PCF Redis tile is just single instance Redis. We need use the Redis cluster broker to import the existing Redis cluster into the PCF. And on this tutorial, we ony focus on how setup an OSS Redis Cluster.

The offical website is here: https://redis.io/

The following demo shows how to setup Redis Cluster on a Linux machine with docker and brief introduction about the Redis Cluter.

Please note: This demo only works for the Linux.

## Setup Redis Cluster

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
```Successfully built 3e40559cd5f7
Successfully tagged iceshi/redis:4.0.9
ice@ice-ubuntu:~/redis/rediscluster$ 
```

Create the redis.conf file:
```port 7000 ## change port number for each instance
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
root@ice-ubuntu:/home/ice/redis/rediscluster# netstat -tnlp | grep redis
tcp        0      0 0.0.0.0:7000            0.0.0.0:*               LISTEN      13426/redis-server
tcp        0      0 0.0.0.0:17000           0.0.0.0:*               LISTEN      13426/redis-server
```



After all instances be started, we can use the redis cli to manage the Cluster.
```sudo apt-get install redis-tools```

Then connect to Redis:
```redis-cli -p 7000```

Please check the "OSS Redis Brief Introduction" for more details.


