# Image Pull Optimization with stargz snapshotter 

This repo will outline the steps to use stargz snapshotter with k3s on a 
rasberrry pi cluster and the benefits of using the same

Stargz snapshotter is a snapshotter implementation for containerd, for more details 
please refer the below link

https://github.com/containerd/stargz-snapshotter

## Set Up

This set up will be on arm64 cluster

### 1. Install Fuse

This snapshotter leverages [crfs](https://github.com/google/crfs) hence the 
dependency on fuse

```shell
apt-get update -y && apt-get install --no-install-recommends -y fuse
```

### 2. stargz-snapshotter

This daemon needs to be running in every node of the cluster

For amd-64 image , please fetch from the release folder of the stargz snapshotter repo.

for arm 64 image , download from this repo

```shell
git clone https://github.com/itsmurugappan/stargz-snapshotter-k3s.git && \
cd stargz-snapshotter-k3s/files && \
sudo mv config/etc/containerd-stargz-grpc /etc/ && \
sudo mv stargz-snapshotter.service /etc/systemd/system/ && \
sudo gunzip stargz-snapshotter-v0.2.0-linux-arm64.tar.gz || true && \
sudo tar -xvf stargz-snapshotter-v0.2.0-linux-arm64.tar && \
sudo chmod +x -R out/ && \
sudo mv out/* /usr/local/bin/ && \
sudo systemctl enable stargz-snapshotter && \
sudo systemctl start stargz-snapshotter
```

### 3. k3s

Please fetch the latest k3s (1.19.x onwards) to get the containerd newer than 1.4.1 which is a pre req for using this.

once you start the k3s like below
```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -s -
```
Get the cni bin directory

```shell
sudo ls /var/lib/rancher/k3s/data/

c302b35fc6551b29d5730cd6fd02747d3b063b3930cf16c090aedcbcc96c45d9  current
```
Replace that in the [tmpl file](./files/config.toml.tmpl#L19)

move the files to k3s config directory and restart k3s

```shell
sudo mv files/config.toml.tmpl /var/lib/rancher/k3s/agent/etc/containerd/
# master
sudo systemctl restart k3s
# agent
sudo systemctl restart k3s-agent
```

Set up is complete on master, please do the same on all worker nodes

### Testing

Create the below pods to check the start up times.

```
# stargz image
apiVersion: v1
kind: Pod
metadata:
  name: gohw-estgz
spec:
  containers:
  - name: gohw-estgz
    image: ghcr.io/itsmurugappan/go-hw-estgz:v1
    ports:
    - containerPort: 8080 

# plain image
apiVersion: v1
kind: Pod
metadata:
  name: gohw
spec:
  containers:
  - name: gohw
    image: ghcr.io/itsmurugappan/go-hw:latest
    ports:
    - containerPort: 8080 
```
* Plain image should start in 2 minutes in a rpi cluster
* Stargz image should start in 20 seconds in a rpi cluster
* Both images are around 700 mb size

### Further steps

You can take it forward by installing knative and see how it improves the cold starts.

https://itsmurugappan.medium.com/knative-on-raspberry-pi-1106984de5b8

### Building estargz image

For building the estargz image you need the ctr-remote client which would be installed as part of step number [2](#2-stargz-snapshotter)

```
# docker login
docker login ghcr.io

# below is the way to optimize a java non estgz image
sudo ctr-remote image optimize  --reuse  --entrypoint='["java", "-Djava.security.egd=file:/dev/./urandom", "-jar", "/helloworld.jar"]' \
           ghcr.io/itsmurugappan/java-hw:latest \
           ghcr.io/itsmurugappan/java-hw-estgz
```
