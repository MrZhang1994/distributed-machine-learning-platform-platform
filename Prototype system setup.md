# Kubernetes Deployment

## Prerequisites

For the master and edge node:

### Installing runtime (docker), `kubeadm`, `kubelet`, `kubectl`

Note: the version installed here must meet the version of the Kubernetes control plane that will be installed by `kubeadm`.

version used:

- Docker: 19.03.12

- kubectl / kubelet / kubeadm: v1.19.0

To install  `kubeadm`, `kubelet`, `kubectl`, 

```bash
# make apt support ssl
apt-get update && apt-get install -y apt-transport-https
# get gpg 
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
# add mirror source of k8s 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
# update 
apt-get update
apt-get install -y kubelet kubeadm kubectl
```

### Check cgroup driver 

Docker and kubelet need the same cgroup driver. To modify docker’s, change in file `/etc/docker/daemon.json` as `"exec-opts": ["native.cgroupdriver=cgroupfs"],` and check by `docker info | grep Cgroup`. Then restart docker `systemctl restart docker`

For kubelet, modify `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`, add 

```
`Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --cgroup-driver=cgroupfs"`
```

and restart kubelet 

```bash
systemctl daemon-reload
systemctl restart kubelet
```

### Turn swap off 

- Temporary turn off: run `swapoff -a`, disable immediately 
- Always turn off: modify file `/etc/fstab`, comment the second line such that 

<img src="/Users/yuxinmiao/Library/Application Support/typora-user-images/image-20200920190606773.png" alt="image-20200920190606773" style="zoom:50%;" />

​		Then `reboot`.

To check, through the command `top`, as `KiB Swap` all zero .

## Initialization on master

This part only for master node.  

### Pull Image

As the images needed could not be pullde directly, first need to pull using mirror source, with specified version, which correspond to kubernetes vresion v1.18.3

```bash
docker pull mirrorgooglecontainers/kube-apiserver:v1.13.2
docker pull mirrorgooglecontainers/kube-controller-manager:v1.13.2
docker pull mirrorgooglecontainers/kube-scheduler:v1.13.2
docker pull mirrorgooglecontainers/kube-proxy:v1.13.2
docker pull mirrorgooglecontainers/pause:3.1
docker pull mirrorgooglecontainers/etcd:3.2.24
docker pull coredns/coredns:1.2.6
```

Then tag the corresponding images

```bash
docker tag mirrorgooglecontainers/kube-apiserver:v1.13.2 101.132.34.209:80/kube-apiserver:v1.13.2
docker tag mirrorgooglecontainers/kube-proxy:v1.13.2 101.132.34.209:80/kube-proxy:v1.13.2
docker tag mirrorgooglecontainerso/kube-controller-manager:v1.13.2 101.132.34.209:80/kube-controller-manager:v1.13.2
docker tag mirrorgooglecontainers/kube-scheduler:v1.13.2 101.132.34.209:80/kube-scheduler:v1.13.2
docker tag mirrorgooglecontainers/coredns:1.2.6 101.132.34.209:80/coredns:1.2.6
docker tag mirrorgooglecontainers/etcd:3.2.24 101.132.34.209:80/etcd:3.2.24
docker tag mirrorgooglecontainers/pause:3.1 101.132.34.209:80/pause:3.1
docker tag mirrorgooglecontainers/kubernetes-dashboard-amd64:v1.10.0 101.132.34.209:80/kubernetes-dashboard-amd64:v1.10.0
```

### kubeadm init

To apply pod network  `flannel` to the cluster, `--pod-network-cidr=10.244.0.0/16` is needed.

run on master 

```bash
sudo kubeadm init \
--pod-network-cidr=10.244.0.0/16 \
--kubernetes-version v1.18.3
```

When see 

```bash
Your Kubernetes control-plane has initialized successfully!
  
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.68:6443 --token 88mamu.oed0ryk6q1ln99im \
    --discovery-token-ca-cert-hash sha256:6d3ad6f127b16420025bba5c91e409d05b332d668e3a8dad4c2a7de6d2f5752f
```

The initialization succeed. Remember save the output following `kubeadm join`.

- Troubleshooting:

  - `docker` or `kubelet` is not running. Check status: `systemctl status [docker|kubelet]`. To start: use `systemctl start [docker|kubelet] `.

  - To reinitiate it, first run `kubeadm reset` and the folowing command

    ```bash
    systemctl stop kubelet
    systemctl stop docker
    rm -rf /var/lib/cni/
    rm -rf /var/lib/kubelet/*
    rm -rf /etc/cni/
    ifconfig cni0 down
    ifconfig flannel.1 down
    ifconfig docker0 down
    ip link delete cni0
    ip link delete flannel.1
    systemctl start docker
    systemctl start kubelet
    ```

    othereise reinitialization will not work. 

  - Use `kubectl get pods -n kube-system` to check pods’ status. All should show running. To see the error in detail, use `kubectl describe node [pod's name] -n kube-system`. The error might caused by the image pulling and the name could be seen from `describe` command. Download it and tranfer to the node. 

Before next step, first following the instruction, do

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- Token valid time 

  token will only be valid for next 24 hours. If after 24 hours, still need to join nodes, run 

  ```bash
  kubeadm token create	# generate new token 
  kubeadm token list		# replace the new token into the join command
  ```

### kubectl configration

```bash
# run the following code 
mkdir -p /root/.kube && \
cp /etc/kubernetes/admin.conf /root/.kube/config

# check kubectl
kubectl get nodes	# show master node
kubectl get cs		# show cluster status 
```

### flannel configration

Run

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
```

- If could not get the file, download the file then transfer to master. 

When shows 

```bash
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds-amd64 created
daemonset.extensions/kube-flannel-ds-arm64 created
daemonset.extensions/kube-flannel-ds-arm created
daemonset.extensions/kube-flannel-ds-ppc64le created
daemonset.extensions/kube-flannel-ds-s390x created
```

Then completed. Also could check through `kubectl get pods -n kube-system` to see the pod status. 

## Join the node 

Node should meet the prerequisites mentioned at first. Run the output saved before 

```bash
kubeadm join 192.168.0.68:6443 --token 88mamu.oed0ryk6q1ln99im \
    --discovery-token-ca-cert-hash sha256:6d3ad6f127b16420025bba5c91e409d05b332d668e3a8dad4c2a7de6d2f5752f
```

when output

```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.
Run 'kubectl get nodes' on the master to see this node join the cluster.
```

the node has joined the cluster. Check on master with `kubeadm get nodes`, which will show two nodes with status ready. 

