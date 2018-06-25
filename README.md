# crio-howto
Just a readme on how to install crio-o with runc and katacontainers

This tutorial uses a VM with ubuntu 16.04

``` bash

##
# Prepare system for k8s install
##

## Add repos for kubeadm/kubelet/kubectl
apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

## Add repo for crio/runc/crictl
add-apt-repository -y ppa:projectatomic/ppa

## Add kata containers repo
sh -c "echo 'deb http://download.opensuse.org/repositories/home:/katacontainers:/release/xUbuntu_$(lsb_release -rs)/ /' > /etc/apt/sources.list.d/kata-containers.list"
curl -sL  http://download.opensuse.org/repositories/home:/katacontainers:/release/xUbuntu_$(lsb_release -rs)/Release.key | apt-key add -

apt-get update

# apt-get install -y kata-runtime kata-proxy kata-shim
apt-get install -y kubelet kubeadm kubectl cri-o-1.10 cri-tools-1.10


## Setup kubelet with crio as runtime
cat > /etc/systemd/system/kubelet.service.d/20-cri.conf <<EOF
[Service]
Environment="KUBELET_EXTRA_ARGS=--container-runtime=remote --runtime-request-timeout=15m --image-service-endpoint unix:///var/run/crio/crio.sock --container-runtime-endpoint unix:///var/run/crio/crio.sock"
EOF

## Fix some crio deprecation bugs...
echo "runtime-endpoint: unix:///var/run/crio/crio.sock" > /etc/crictl.yaml 
## Set kata as trusted runtime
sed -i '/^runtime_untrusted_workload/c\runtime_untrusted_workload = "/usr/bin/kata-runtime"' /etc/crio/crio.conf
## Set cgroupfs as cgroup manager
sed -i '/^cgroup_manager/c\cgroup_manager = "cgroupfs"' /etc/crio/crio.conf


systemctl daemon-reload
systemctl enable crio kubelet
systemctl start crio kubelet

##
# Setup kubernetes with kubeadm (single node)
##
kubeadm init --skip-preflight-checks --cri-socket /var/run/crio/crio.sock --pod-network-cidr=10.244.0.0/16

echo "Copying kubeconfig..."
mkdir -p /root/.kube
cp -i /etc/kubernetes/admin.conf /root/.kube/config

## Allow workload on kubernetes master
echo "Tainting master node..."
kubectl taint nodes --all node-role.kubernetes.io/master-

## Deploy an echoserver
echo "Deploying echoserver..."
kubectl run echoserver \
--image=gcr.io/google_containers/echoserver:1.8 \
--port=8080 \
--replicas=2

## Expose an echoserver
kubectl expose deployment echoserver --type=NodePort --port=8080 --target-port=31080
```
