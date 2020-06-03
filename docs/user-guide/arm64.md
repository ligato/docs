# ARM64

---

This section outlines the procedures for installing an ARM64-compatible version of the VPP agent. The same procedures described in the the [Quickstart](quickstart.md) and [VPP Agent Setup](get-vpp-agent.md) sections of the User Guide apply here, except ARM64 images are used.  

---

## ARM64 Docker Image Pull

Pre-built production and development images supporting ARM64 are available from [Dockerhub](https://hub.docker.com/u/ligato).

### Production Image

** Included in the pre-built ARM64 production image:**

- Binaries for the VPP agent with default config files
- Installation of the compatible VPP

Pull the pre-built ARM64 production image:
```json
docker pull ligato/vpp-agent-arm64
```

### Development Image

**Included in the pre-built ARM64 development image:**

- VPP agent with default config files including source code
- Compatible VPP installation including source code and build artifacts
- Development environment with all requirements to build the VPP agent and the VPP
- Tools to generate code (proto models, binary APIs)

Pull the pre-built ARM64 development image:
```json
docker pull ligato/dev-vpp-agent-arm64
```
---

## Start VPP Agent

Start Production Image
```json
docker run -it --rm --name vpp-agent -p 5002:5002 -p 9191:9191 --privileged ligato/vpp-agent-arm64
```
Start Development Image
```json
docker run -it --rm --name vpp-agent -p 5002:5002 -p 9191:9191 --privileged ligato/dev-vpp-agent-arm64
```

Check that the vpp-agent is running using [Agentctl](agentctl.md)

Agentctl help:
```
docker exec -it vpp-agent agentctl --help
```
Agentctl Status:
```json
docker exec -it vpp-agent agentctl status
```

---

## Start etcd

!!! note
    Check for the proper etcd ARM64 docker image in the [official repository][etcd]. Currently, you must use the parameter `-e ETCD_UNSUPPORTED_ARCH=arm64`.

The following command starts etcd for ARM64 in a docker container. If the image is not present on your local machine, docker will download it first.
```
sudo docker run -p 2379:2379 --name etcd -e ETCDCTL_API=3 -e ETCD_UNSUPPORTED_ARCH=arm64 \
    quay.io/coreos/etcd:v3.3.8-arm64 /usr/local/bin/etcd \
    -advertise-client-urls http://0.0.0.0:2379 \
    -listen-client-urls http://0.0.0.0:2379
```

Use etcdctl to verify the etcd server is running:
```json
docker exec -it etcd etcdctl endpoint health
```
!!! Note
    Agentctl uses `127.0.0.1:2379` as the default address of the etcd server. If the etcd server is running in a separate container, then its configured address must be passed to agentctl as an argument. This is accomplished using the `-e` or `--etcd-endpoints` flags. 

Use the agentctl kvdb list command to show the contents of the etcd data store:
```json
docker exec -it vpp-agent agentctl -e 172.17.0.1:2379 akvdb list
```
Alternatively, you can use [etcdctl](quickstart.md#51-etcdctl) to interact with the etcd data store. 

---

## Kafka and ARM64

There is no official spotify/kafka image for the ARM64 platform. You can build an image following the steps contained in the [repository][kafka]. 

However, you will need to modify the Kafka/Dockerfile before building the image like so:
```
//FROM java:openjdk-8-jre
FROM openjdk:8-jre
...
// ENV KAFKA_VERSION 0.10.1.0
ENV KAFKA_VERSION 0.10.2.1
...
```
For preparing the kafka-arm64 image, use:

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

[agentctl]: ../user-guide/agentctl.md
[dockerhub]: https://hub.docker.com/r/ligato/vpp-agent-arm64/
[etcd]: https://quay.io/repository/coreos/etcd?tag=latest&tab=tags
[kafka]: https://github.com/spotify/docker-kafka#build-from-source
[ligato-arm64-image]: https://hub.docker.com/r/ligato/dev-vpp-agent-arm64/
[ligato-arm64-image-tags]: https://hub.docker.com/r/ligato/dev-vpp-agent-arm64/tags/
