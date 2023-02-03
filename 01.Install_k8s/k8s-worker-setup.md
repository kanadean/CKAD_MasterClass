# Setup Kubernetes worker

Follow instructions in this section to setup kubernetes worker node.
The instuctions are written for Ubuntu 20.04 and k8s 1.26.

Steps to install master node,
- Set kubernetes and OS version IDs
- Verify pre-requisites and turn SWAP Off
- Cleanup existing k8s Install 
- Download and Install Required Packages
- Setup various config files
- Obtain token from master node
- Join the node to the cluster
- Optionally Configure bash for aliases etc

## Set kubernetes and OS version IDs (as root)
```
sudo su
K8S_VER=1.26.1
OS_VER=20.04
```

## Verify pre-requisites and turn SWAP Off (as root)
```
# DISABLE the SWAP
swapoff -a
# Comment the swap line in fstab to prevent auto mounting of SWAP on reboot
sed -i '/\sswap\s/ s/^\(.*\)$/#\1/g' /etc/fstab
```

## Cleanup existing k8s Install (as root)
```
### remove packages
kubeadm reset -f || true
rm -rf /opt/cni/bin
crictl rm --force $(crictl ps -a -q) || true
apt-mark unhold kubelet kubeadm kubectl kubernetes-cni || true
apt-get remove -y containerd kubelet kubeadm kubectl kubernetes-cni || true
apt-get autoremove -y
systemctl daemon-reload
```

## Download and Install Required Packages (as root)
```
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${OS_VER}/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:testing.list
curl -L "https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${OS_VER}/Release.key" | sudo apt-key add -
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get -y install cri-tools containers-common
rm /etc/apt/sources.list.d/devel:kubic:libcontainers:testing.list
apt-get install -y containerd kubelet=${K8S_VER}-00 kubeadm=${K8S_VER}-00 kubectl=${K8S_VER}-00 kubernetes-cni
apt-mark hold kubelet kubeadm kubectl kubernetes-cni

```
### Remove latest containerd (it is buggy) and replace with older one
(REF: https://forum.linuxfoundation.org/discussion/862825/kubeadm-init-error-cri-v1-runtime-api-is-not-implemented)
```
### install containerd 1.6 over apt-installed-version
wget https://github.com/containerd/containerd/releases/download/v1.6.12/containerd-1.6.12-linux-amd64.tar.gz
tar xvf containerd-1.6.12-linux-amd64.tar.gz
systemctl stop containerd
mv bin/* /usr/bin
rm -rf bin containerd-1.6.12-linux-amd64.tar.gz
systemctl unmask containerd
systemctl start containerd
```

## Setup various config files (as root)
```
### containerd
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
sudo mkdir -p /etc/containerd


### containerd config
cat > /etc/containerd/config.toml <<EOF
disabled_plugins = []
imports = []
oom_score = 0
plugin_dir = ""
required_plugins = []
root = "/var/lib/containerd"
state = "/run/containerd"
version = 2
[plugins]
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
      base_runtime_spec = ""
      container_annotations = []
      pod_annotations = []
      privileged_without_host_devices = false
      runtime_engine = ""
      runtime_root = ""
      runtime_type = "io.containerd.runc.v2"
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        BinaryName = ""
        CriuImagePath = ""
        CriuPath = ""
        CriuWorkPath = ""
        IoGid = 0
        IoUid = 0
        NoNewKeyring = false
        NoPivotRoot = false
        Root = ""
        ShimCgroup = ""
        SystemdCgroup = true
EOF


### crictl uses containerd as default
{
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
EOF
}


### kubelet should use containerd
{
cat <<EOF | sudo tee /etc/default/kubelet
KUBELET_EXTRA_ARGS="--container-runtime remote --container-runtime-endpoint unix:///run/containerd/containerd.sock"
EOF
}


### start services
systemctl daemon-reload
systemctl enable containerd
systemctl restart containerd
systemctl enable kubelet && systemctl restart kubelet
```

## JOIN Worker node to Cluster

### Print JOIN Command from master node (EXECUTE ON MASTER NODE)
```
kubeadm token create --print-join-command --ttl 0
```

### Copy the output of JOIN and execute on worker node
```
kubeadm join .............
```

## Verify Cluster Status (Execute on master)
```
kubectl get nodes
```
