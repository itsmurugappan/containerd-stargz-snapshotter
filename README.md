# Image Pull Optimization with stargz snapshotter on k3s

This repo will outline the steps to use stargz snapshotter with k3s on a 
rasberrry pi cluster and the benefits of using the same

Stargz snapshotter is a snapshotter implementation for containerd, for more details 
please refer the below link

https://github.com/containerd/stargz-snapshotter

## Set Up

This set up will be on arm64 cluster with Ubuntu (version: Ubuntu 20.04.1 LTS)

### stargz-snapshotter

This daemon needs to be running in every node of the cluster. Choose between nightly and release

#### Release

```shell
export tag=v0.4.1

pushd $(pwd) && \
sudo mkdir -p stargz-snapshotter-k3s/files && cd stargz-snapshotter-k3s && \
sudo curl -LO https://github.com/containerd/stargz-snapshotter/releases/download/$tag/stargz-snapshotter-$tag-linux-arm64.tar.gz && \
cd files && \
sudo curl -LO https://raw.githubusercontent.com/itsmurugappan/containerd-stargz-snapshotter/main/files/config.toml.tmpl && \
sudo curl -LO https://raw.githubusercontent.com/itsmurugappan/containerd-stargz-snapshotter/main/files/stargz-snapshotter.service && 
sudo curl -LO https://raw.githubusercontent.com/itsmurugappan/containerd-stargz-snapshotter/main/files/config/etc/containerd-stargz-grpc/config.toml && \
sudo mkdir -p /etc/containerd-stargz-grpc && \
sudo mv config.toml /etc/containerd-stargz-grpc/ && \
sudo mv stargz-snapshotter.service /etc/systemd/system/ && \
cd .. && \
sudo gunzip stargz-snapshotter-$tag-linux-arm64.tar.gz || true && \
sudo tar -xvf stargz-snapshotter-$tag-linux-arm64.tar && \
sudo chmod +x containerd-stargz-grpc ctr-remote && \
sudo mv ctr-remote containerd-stargz-grpc /usr/local/bin/ && \
sudo systemctl enable stargz-snapshotter && \
sudo systemctl restart stargz-snapshotter
```

#### Nightly
For nightly arm 64 image , download from this repo.

```shell
pushd $(pwd) && \
git clone https://github.com/itsmurugappan/stargz-snapshotter-k3s.git && \
cd stargz-snapshotter-k3s/files && \
sudo mkdir -p /etc/containerd-stargz-grpc && \
sudo mv config/etc/containerd-stargz-grpc/config.toml /etc/containerd-stargz-grpc/ && \
sudo mv stargz-snapshotter.service /etc/systemd/system/ && \
cd ./.. && \
sudo gunzip releases/stargz-snapshotter-nightly-linux-arm64.tar.gz || true && \
sudo tar -xvf releases/stargz-snapshotter-nightly-linux-arm64.tar && \
sudo chmod +x -R out/ && \
sudo mv out/* /usr/local/bin/ && \
sudo systemctl enable stargz-snapshotter && \
sudo systemctl restart stargz-snapshotter && \
cd ..
```

### k3s

Please fetch the latest k3s (1.19.x onwards) to get the containerd newer than 1.4.1 which is a pre req for using this.

once you start the k3s like below
```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -s -
```

move the containerd config files

```shell
sudo mv files/config.toml.tmpl /var/lib/rancher/k3s/agent/etc/containerd/
# master
sudo systemctl restart k3s
# agent
sudo systemctl restart k3s-agent
```

Set up is complete on master, please do the same on all worker nodes

#### cleanup
```shell
popd && \
sudo rm -rf stargz-snapshotter-k3s
```

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
    image: ghcr.io/itsmurugappan/hw:v3
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
sudo ctr-remote image optimize  --oci --reuse  --entrypoint='["java", "-Djava.security.egd=file:/dev/./urandom", "-jar", "/helloworld.jar"]' \
           ghcr.io/itsmurugappan/java-hw:latest \
           ghcr.io/itsmurugappan/java-hw-estgz
```
