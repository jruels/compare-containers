# Migrate Kubernetes from Docker to CRI-O

# Overview
This lab explains how to convert a running Kubernetes cluster from Docker to CRI-O.

## Confirm runtime
Run the following on the master node:

Start by confirming the runtime of the cluster.
```
kubectl show nodes -o wide
```

Output will be similar to: 
```
NAME              STATUS   ROLES                  AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
ip-10-0-102-201   Ready    control-plane,master   26m   v1.21.0   10.0.102.201   <none>        Ubuntu 20.04.2 LTS   5.4.0-1045-aws   docker://20.10.6
ip-10-0-102-47    Ready    <none>                 21m   v1.21.0   10.0.102.47    <none>        Ubuntu 20.04.2 LTS   5.4.0-1045-aws   docker://20.10.6
ip-10-0-102-56    Ready    <none>                 21m   v1.21.0   10.0.102.56    <none>        Ubuntu 20.04.2 LTS   5.4.0-1045-aws   docker://20.10.6
```
As you can see all of the nodes in the cluster are using Docker as the container runtime. 






## Migrate to CRI-O

Before changing the container runtime, we need to migrate all containers to another node using the `drain` and `cordon` commands. 

First, choose a node that does **NOT** have the `control-plane` role: 
```
kubectl get nodes
```

Output: 
```
NAME              STATUS   ROLES                  AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
ip-10-0-102-201   Ready    control-plane,master   26m   v1.21.0   10.0.102.201   <none>        Ubuntu 20.04.2 LTS   5.4.0-1045-aws   docker://20.10.6
ip-10-0-102-47    Ready    <none>                 21m   v1.21.0   10.0.102.47    <none>        Ubuntu 20.04.2 LTS   5.4.0-1045-aws   docker://20.10.6
ip-10-0-102-56    Ready    <none>                 21m   v1.21.0   10.0.102.56    <none>        Ubuntu 20.04.2 LTS   5.4.0-1045-aws   docker://20.10.6
```

### Cordon Kubernetes node

Cordon the node so the scheduler ignores it. Remember to replace the example node with a node from your cluster. 
```
kubectl cordon ip-10-0-102-56 
```

### Drain Kubernetes node

After cordoning the node, use `drain` to stop all running containers. These containers will be rescheduled to another node with available resources.
```
kubectl drain ip-10-0-102-56 --ignore-daemonsets
```
`--ignore-daemonsets` tells Kubernetes not to cordon `kube-proxy`, `weave` or other daemonsets. If you do not pass this argument the `cordon` command fails.

Run the following on the node you cordoned and drained.
### Stop services

Stop the services on the node.
```
sudo systemctl stop kubelet docker
```
## Install CRI-O

Set some environment variables to install the correct version of CRI-O
* `OS` = Operating System version 
* `VERSION` = Kubernetes version

```
export OS=xUbuntu_20.04
export VERSION=1.21
```
Download and install CRI-O
```
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list

curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | apt-key add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | apt-key add -

apt-get update
apt-get install -y cri-o cri-o-runc
```

Set some system control settings and start `crio` daemon:
```
cat << EOF > /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system
systemctl enable crio
systemctl start crio
```

At this point you should be able to run `crictl info` and `crictl ps` successfully.

### Change container runtime
Update the `kubelet` configuration to use `crio`   
Edit the file `/var/lib/kubelet/kubeadm-flags.env`. 

Add the `crio` runtime to the flags. `--container-runtime=remote` and `--container-runtime-endpoint=unix:///var/run/crio/crio.sock`




The `/var/lib/kubelet/kubeadm-flags.env` file should now look like this

```
KUBELET_KUBEADM_ARGS="--network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.4.1 --container-runtime=remote --container-runtime-endpoint=unix:///var/run/crio/crio.sock --runtime-request-timeout=10m --cgroup-driver=systemd"
```
### Start kubelet
After changing the runtime start the kubelet service.
```
sudo systemctl start kubelet
```

### Confirm CRI-O runtime
Run the following on the master node:
Now confirm the node is using the `crio` runtime.
```
kubectl get nodes -o wide
```

Output: 
```
NAME              STATUS                     ROLES                  AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
ip-10-0-102-201   Ready                      control-plane,master   49m   v1.21.0   10.0.102.201   <none>        Ubuntu 20.04.2 LTS   5.4.0-1045-aws   docker://20.10.6
ip-10-0-102-47    Ready,SchedulingDisabled   <none>                 45m   v1.21.0   10.0.102.47    <none>        Ubuntu 20.04.2 LTS   5.4.0-1045-aws   docker://20.10.6
ip-10-0-102-56    Ready                      <none>                 45m   v1.21.0   10.0.102.56    <none>        Ubuntu 20.04.2 LTS   5.4.0-1045-aws   cri-o://1.21.0
```
As you can see node `ip-10-0-102-56` is running `cri-o` in my configuration.

### Uncordon node 
The node changed to use `containerd` is still cordoned. To add it back into the scheduling pool it must be uncordoned. 
```
kubectl uncordon ip-10-0-102-56
```

Output: 
```
node/ip-10-0-102-56 uncordoned
```

Confirm scheduling is no longer disabled.
```
kubectl get nodes
```

Output: 
```
NAME             STATUS   ROLES    AGE     VERSION
ip-10-0-102-56   Ready    <none>   3h50m   v1.21.0
```

Now deploy a demo application to confirm scheduling to all the nodes works 

```
kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
```

Confirm the pods are deployed onto the worker nodes
```
kubectl get pods -o wide 
```

Output: 
```
NAME                                READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES
nginx-deployment-66b6c48dd5-2hh6k   1/1     Running   0          20s   10.36.0.1   ip-10-0-102-56   <none>           <none>
nginx-deployment-66b6c48dd5-hq46m   1/1     Running   0          20s   10.44.0.1   ip-10-0-102-47   <none>           <none>
nginx-deployment-66b6c48dd5-z9gvc   1/1     Running   0          20s   10.44.0.2   ip-10-0-102-47   <none>           <none>
```

Log into the worker node and confirm the `crio` runtime was used to create the containers.
```
sudo crictl pods
```

Output: 
```
POD ID              CREATED             STATE               NAME                                NAMESPACE           ATTEMPT
ee668c0a654fd       3 minutes ago       Ready               nginx-deployment-66b6c48dd5-2hh6k   default             0
932cf76cc2a27       22 minutes ago      Ready               kube-proxy-x54nq                    kube-system         0
7683edcadb531       22 minutes ago      Ready               weave-net-tkd2d                     kube-system         0
```


We can see that the `nginx` containers were created with `crio`

Now that this node has been successfully migrated to crio, migrate the other worker node using the same steps. 
# Congrats! 
