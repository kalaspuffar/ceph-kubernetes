# ceph-kubernetes

An example of how to setup ceph mounting on a kubernetes cluster.

### Ceph Packages

First up after you have an debian machine up and running you could install ceph so that is available to your cluster. This is the standard setup to get the pacific packages.

```
wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -
echo deb https://download.ceph.com/debian-pacific/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
sudo apt update
sudo apt install ceph
```

### Kubernetes cluster

I follow this really interesting guide to setup a kubernetes cluster from [redpill](https://www.redpill-linpro.com/techblog/2019/04/04/kubernetes-setup.html)


#### Enable net.bridge.bridge-nf-call-iptables
This is required by Flannel and possibly other networking options. You can read more at https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network

```
cat > /etc/sysctl.d/20-bridge-nf.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

#### A kubernetes cluster don't like swap

You could either comment out your swap drives in `/etc/fstab` or just run swapoff every boot.
```
swapoff -a
```

#### Install Docker with recommended settings

```
mkdir /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
apt-get update
apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2
curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
echo 'deb [arch=amd64] https://download.docker.com/linux/debian stretch stable' > /etc/apt/sources.list.d/docker.list
apt-get update
apt-get install -y --no-install-recommends docker-ce
```

The --no-install-recommends will avoid pulling in stuff you don’t need, including the aufs DKMS package.

#### Install Kubernetes components

```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
```

The xenial in the APT source is correct. That’s the repo they seem to update, and these are Go binaries anyway so they’re self-contained.

#### Run kubeadm to set up the cluster

The --pod-network-cidr setting is required by Flannel, which I chose to use for pod networking.

```
kubeadm init --pod-network-cidr=10.244.0.0/16
```
That’s it. Neat, huh?

There is still a bunch of work to do to make the cluster actually useful. You can do most of the rest of this as a non-root user. Follow the instructions kubeadm gave you to copy the credential as your regular user.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

And as a handy extra tip, you’ll want completion:

```
source <(kubectl completion bash)
```

####  Install Flannel for pod networking
```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
kubectl get pods --all-namespaces
```

You should see coredns pods come to life if all is well.

#### Untaint the master so you can run pods
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### Setting up secret and adding pod.

First of you need to setup a new user on your cluster. The command below setup a new client named kubernetes that could read and write to the kubernetes directory in the ceph filesystem called cephfs.
```
sudo ceph fs authorize cephfs client.kubernetes /kubernetes rw
```

This will give you a key. Then you need to add that to the file `ceph-secret.yml` in this repository.
```
stringData:
  key: AQCasDxhaaaaMRAAI0000TEZxyTzb+B7777r4A==
```

Next up you need to look at the file `ceph-pod.yml`. Things you might want to modify is the image to pull, the directory you want to mount and mount too. And perhaps the name of the user or monitor setup.

In order to set these up in the cluster it is as easy as applying them.

```
kubectl apply -f ceph-secret.yml
kubectl apply -f ceph-pod.yml
```

You are now up and running.

If you want to try it out you just run the command below and change out `ceph-example` with the pod name you chose.

```
kubectl exec --stdin --tty ceph-example -- /bin/bash
```