# ARM64

---

## Quick start

For a quick start with the VPP Agent, you can use pre-built Docker images with the Agent and VPP on [Dockerhub][dockerhub].

1. Start the ETCD (and optionally Kafka) on your host (see below). 
Note: **The Agent in the pre-built Docker image will not start if it can't connect to both ETCD and Kafka, if used**.

2. Run VPP + VPP Agent in the Docker image:
```
docker pull ligato/vpp-agent-arm64
docker run -it --name vpp --rm ligato/vpp-agent-arm64
```

3. Configure the VPP agent using agentctl:
```
docker exec -it vpp agentctl -h
```

3. Check the configuration (using agentctl or directly using VPP console):
```
docker exec -it vpp agentctl -e 172.17.0.1:2379 show
docker exec -it vpp vppctl -s localhost:5002
```

## AMD64 Images

The VPP Agent can be successfully built also for the ARM64 platform.

### Development images

For a quick start with the development image, you can download the [official image for ARM64 platform][ligato-arm64-image] from **DockerHub**.

```sh
// latest release (stable)
$ docker pull docker.io/ligato/dev-vpp-agent-arm64

// bleeding edge (unstable)
$ docker pull docker.io/ligato/dev-vpp-agent-arm64:pantheon-dev	# 
```

List of all available docker image tags for development image can be found [here for ARM64][ligato-arm64-image-tags].

### Production images
For a quick start with the VPP Agent, you can use pre-build Docker images with the Agent and VPP on Dockerhub:
the [official image for ARM64 platform][ligato-arm64-image].
```
docker pull ligato/vpp-agent-arm64
```

### ARM64 and ETCD Server

You can run the ETCD server in a separate container on your local host as follows:
```
sudo docker run -p 2379:2379 --name etcd -e ETCDCTL_API=3 -e ETCD_UNSUPPORTED_ARCH=arm64 \
    quay.io/coreos/etcd:v3.3.8-arm64 /usr/local/bin/etcd \
    -advertise-client-urls http://0.0.0.0:2379 \
    -listen-client-urls http://0.0.0.0:2379
```

The ETCD server will be available on your host OS IP `172.17.0.1` (by default) and port `2379`. Call the agent via ETCD using the testing client:
```
vpp-agent-ctl /opt/vpp-agent/dev/etcd.conf -tap
vpp-agent-ctl /opt/vpp-agent/dev/etcd.conf -tapd
```

**Note for ARM64:**

Check for proper etcd ARM64 docker image in the [official repository][etcd]. Currently you must use the parameter `-e ETCD_UNSUPPORTED_ARCH=arm64`.

### ARM64 and Kafka

There is no official spotify/kafka image for ARM64 platform. You can build an image following steps at the [repository][kafka]. However you need to modify the kafka/Dockerfile before building like this:
```
//FROM java:openjdk-8-jre
FROM openjdk:8-jre
...
// ENV KAFKA_VERSION 0.10.1.0
ENV KAFKA_VERSION 0.10.2.1
...
```
For preparing kafka-arm64 image use:

```
git clone https://github.com/spotify/docker-kafka.git
cd docker-kafka
sed -i -e "s/FROM java:openjdk-8-jre/FROM openjdk:8-jre/g" kafka/Dockerfile
sed -i -e "s/ENV KAFKA_VERSION 0.10.1.0/ENV KAFKA_VERSION 0.10.2.1/g" kafka/Dockerfile
docker build -t kafka-arm64 kafka/
```

Then you can start Kafka in a separate container:
```
sudo docker run -p 2181:2181 -p 9092:9092 --name kafka --rm \
 --env ADVERTISED_HOST=172.17.0.1 --env ADVERTISED_PORT=9092 kafka-arm64
```

[dockerhub]: https://hub.docker.com/r/ligato/vpp-agent-arm64/
[etcd]: https://quay.io/repository/coreos/etcd?tag=latest&tab=tags
[kafka]: https://github.com/spotify/docker-kafka#build-from-source
[ligato-arm64-image]: https://hub.docker.com/r/ligato/dev-vpp-agent-arm64/
[ligato-arm64-image-tags]: https://hub.docker.com/r/ligato/dev-vpp-agent-arm64/tags/
