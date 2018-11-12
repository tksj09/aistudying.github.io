## 3.1. Master Nodes
Install the required components on your master node:

### 3.1.1. Installing and Running Kubernetes
install docker
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install docker-ce
```
install nvidia-docker2(2.0.3)
```
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey |sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/ubuntu16.04/amd64/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt update
sudo apt-get install nvidia-docker2
sudo pkill -SIGHUP dockerd
```

Add the official GPG keys.
```
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ curl -s -L https://nvidia.github.io/kubernetes/gpgkey | sudo apt-key add -
$ curl -s -L https://nvidia.github.io/kubernetes/ubuntu16.04/nvidia-kubernetes.list |\
           sudo tee /etc/apt/sources.list.d/nvidia-kubernetes.list
```
Update the package index.   
```
$ sudo apt update
```

Install packages.
```
$ VERSION=1.9.10+nvidia
$ sudo apt install -y kubectl=${VERSION} kubelet=${VERSION} \
                               kubeadm=${VERSION} helm=${VERSION}
```
Start your cluster.
```
$ sudo kubeadm init --ignore-preflight-errors=all --config /etc/kubeadm/config.yml
```
You may choose to save the token and the hash of the CA certificate as part of of kubeadm init to join worker nodes to the cluster later. This will take a few minutes.
For NGC VMIs, issue a chmod command on the newly created file.
```
$ sudo chmod 644 /etc/apt/sources.list.d/nvidia-kubernetes.list
```
### 3.1.2. Checking Cluster Health
Check that all the control plane components are running on the master node:
```
$ kubectl get all --all-namespaces
NAMESPACE     NAME                                       READY     STATUS    RESTARTS   AGE
kube-system   etcd-master                                  1/1     Running   0          2m
kube-system   kube-apiserver-master                        1/1     Running   0          1m
kube-system   kube-controller-manager-master               1/1     Running   0          1m
kube-system   kube-dns-55856cb6b6-c9mvz                    3/3     Running   0          2m
kube-system   kube-flannel-ds-qddkv                        1/1     Running   0          2m
kube-system   kube-proxy-549sh                             1/1     Running   0          2m
kube-system   kube-scheduler-master                        1/1     Running   0          1m
kube-system   nvidia-device-plugin-daemonset-1.9.10-5m95f  1/1     Running   0          2m
kube-system   tiller-deploy-768dcd4457-j2ffs               1/1     Running   0          2m
monitoring    alertmanager-kube-prometheus-0               2/2     Running   0          1m
monitoring    kube-prometheus-exporter-kube-state-52sgg    2/2     Running   0          1m
monitoring    kube-prometheus-grafana-694956469c-xwk6x     2/2     Running   0          1m
monitoring    prometheus-kube-prometheus-0                 2/2     Running   0          1m
monitoring    prometheus-operator-789d76f7b9-t5qqz         1/1     Running   0          2m
```
