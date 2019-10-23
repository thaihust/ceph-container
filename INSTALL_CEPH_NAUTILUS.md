## Build docker image

- Install env to build:

```sh
curl -fsSL https://get.docker.com -o get-docker.sh
chmod +x && bash get-docker.sh
apt-get update && apt-get install -y build-essential
```

- Select nautilus branch: ```git checkout nautilus-14.2.4```

- Build nautilus docker: ```make FLAVORS=nautilus,centos,7 build```

## Manual test without KV store

- Set env:

```sh
DOCKER_IMAGE=ceph/daemon:HEAD-nautilus-centos-7-x86_64
MON_IP=10.240.207.17
CEPH_PUBLIC_NETWORK=10.240.207.0/26
```

- Install ceph-mon:

```
docker rm -f -v ceph-mon
docker run -d --name ceph-mon --net=host \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
-v /sys/fs/cgroup:/sys/fs/cgroup:ro \
-e MON_IP=${MON_IP} \
-e CEPH_PUBLIC_NETWORK=${CEPH_PUBLIC_NETWORK} ${DOCKER_IMAGE} mon
```

- Generate client keyring for osd:

```sh
docker exec -ti ceph-mon ceph auth get client.bootstrap-osd -o /var/lib/ceph/bootstrap-osd/ceph.keyring
```

- Install ceph-mgr:

```sh
docker rm -f -v ceph-mgr
docker run -d --net=host --name ceph-mgr \
--privileged=true \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
${DOCKER_IMAGE} mgr
```

- Add volume and attach to VM. e.g: /dev/vdb

- Install ceph-osd:

  - Zap volume:

  ```sh
  docker rm -f -v ceph-zap-vdb
  docker run -d --name ceph-zap-vdb --privileged=true \
  -v /dev/:/dev/ \
  -v /run/lvm:/run/lvm \
  -e OSD_DEVICE=/dev/vdb \
  ${DOCKER_IMAGE} zap_volume
  ``` 

  - Create ceph-osd container:

  ```sh
  docker rm -f -v ceph-osd-vdb
  docker run -d --name ceph-osd-vdb --net=host \
  --privileged=true \
  --pid=host \
  -v /etc/ceph:/etc/ceph \
  -v /var/lib/ceph/:/var/lib/ceph/ \
  -v /dev/:/dev/ \
  -v /run/lvm:/run/lvm \
  -e OSD_DEVICE=/dev/vdb \
  -e OSD_TYPE=volume \
  -e OSD_BLUESTORE=1 \
  ${DOCKER_IMAGE} osd
  ```
