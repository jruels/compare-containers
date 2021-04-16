# Migrate Kubernetes from Docker to containerd

# Overview
This lab explains how to convert a running Kubernetes cluster from Docker to containerd.

## Confirm runtime
Run the following on the master node:

Start by confirming the runtime of the cluster.
```
kubectl show nodes -o wide
```

Output will be similar to: 
```
NAME              STATUS     ROLES                  AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
ip-10-0-102-138   Ready      control-plane,master   2m25s   v1.21.0   10.0.102.138   <none>        Ubuntu 20.04.2 LTS   5.4.0-1043-aws   docker://20.10.6
ip-10-0-102-27    Ready      <none>                 16s     v1.21.0   10.0.102.27    <none>        Ubuntu 20.04.2 LTS   5.4.0-1043-aws   docker://20.10.6
ip-10-0-102-79    NotReady   <none>                 20s     v1.21.0   10.0.102.79    <none>        Ubuntu 20.04.2 LTS   5.4.0-1043-aws   docker://20.10.6
```
As you can see all of the nodes in the cluster are using Docker as the container runtime. 

## Confirm containerd is installed

Now confirm the containerd cli `ctr` is installed, and list namespaces. 
```
sudo ctr namespace list
```

You should see the `moby` namespace which is created by Docker.

## Migrate to containerd

Before changing the container runtime, we need to migrate all containers to another node using the `drain` and `cordon` commands. 

First, choose a node that does **NOT** have the `control-plane` role: 
```
kubectl get nodes
```

Output: 
```
NAME              STATUS     ROLES                  AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
ip-10-0-102-138   Ready      control-plane,master   2m25s   v1.21.0   10.0.102.138   <none>        Ubuntu 20.04.2 LTS   5.4.0-1043-aws   docker://20.10.6
ip-10-0-102-27    Ready      <none>                 16s     v1.21.0   10.0.102.27    <none>        Ubuntu 20.04.2 LTS   5.4.0-1043-aws   docker://20.10.6
ip-10-0-102-79    NotReady   <none>                 20s     v1.21.0   10.0.102.79    <none>        Ubuntu 20.04.2 LTS   5.4.0-1043-aws   docker://20.10.6
```

### Cordon Kubernetes node

Cordon the node so the scheduler ignores it. Remember to replace the example node with a node from your cluster. 
```
kubectl cordon ip-10-0-102-27
```

### Drain Kubernetes node

After cordoning the node, use `drain` to stop all running containers. These containers will be rescheduled to another node with available resources.
```
kubectl drain ip-10-0-102-27 --ignore-daemonsets
```
`--ignore-daemonsets` tells Kubernetes not to cordon `kube-proxy`, `weave` or other daemonsets. If you do not pass this argument the `cordon` command fails.

Run the following on the node you cordoned and drained.
### Stop services

Stop the services on the node.
```
sudo systemctl stop kubelet docker
```
### Change container runtime

Enable the Container Runtime Interface in `/etc/containerd/config.toml` by commenting the following line: 
```
disabled_plugins = ["cri"]
```
Restart `containerd` to read the new configuration 
```
sudo systemctl restart containerd
```
Update the `kubelet` configuration to use `containerd`
Edit the file `/var/lib/kubelet/kubeadm-flags.env`. 

Add the `containerd` runtime to the flags. `--container-runtime=remote` and `--container-runtime-endpoint=unix:///run/containerd/containerd.sock`

The `/var/lib/kubelet/kubeadm-flags.env` file should now look like this

```
KUBELET_KUBEADM_ARGS="--network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.4.1 --container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
```
### Start kubelet
After changing the runtime start the kubelet service.
```
sudo systemctl start kubelet
```

### Confirm containerd runtime
Run the following on master node:
Now confirm the node is using the `containerd` runtime.
```
kubectl get nodes -o wide
```

Output: 
```
NAME              STATUS                     ROLES                  AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
ip-10-0-102-138   Ready                      control-plane,master   58m   v1.21.0   10.0.102.138   <none>        Ubuntu 20.04.2 LTS   5.4.0-1043-aws   docker://20.10.6
ip-10-0-102-27    Ready                      <none>                 56m   v1.21.0   10.0.102.27    <none>        Ubuntu 20.04.2 LTS   5.4.0-1043-aws   docker://20.10.6
ip-10-0-102-79    Ready,SchedulingDisabled   <none>                 56m   v1.21.0   10.0.102.79    <none>        Ubuntu 20.04.2 LTS   5.4.0-1043-aws   containerd://1.4.4
```
As you can see node `ip-10-0-102-79` is running `containerd` in my configuration.

### Uncordon node 
The node changed to use `containerd` is still cordoned. To add it back into the scheduling pool it must be uncordoned. 
```
kubectl uncordon ip-10-0-102-79
```

Output: 
```
node/ip-10-0-102-79 uncordoned
```

Confirm scheduling is no longer disabled.
```
kubectl get nodes
```

Output: 
```
NAME             STATUS   ROLES    AGE     VERSION
ip-10-0-102-79   Ready    <none>   3h50m   v1.21.0
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
nginx-deployment-66b6c48dd5-6n7k2   1/1     Running   0          19s   10.36.0.1   ip-10-0-102-27   <none>           <none>
nginx-deployment-66b6c48dd5-j5m59   1/1     Running   0          19s   10.44.0.1   ip-10-0-102-79   <none>           <none>
nginx-deployment-66b6c48dd5-q649p   1/1     Running   0          19s   10.44.0.2   ip-10-0-102-79   <none>           <none>
```

Log into the worker node and confirm the `containerd` runtime was used to create the containers.
```
sudo ctr ns ls
```

Output: 
```
NAME   LABELS
k8s.io
moby
```
There are two namespaces, the new `k8s.io` one was created when we enabled `containerd` in the kubelet. 

Check that namespace for running containers 
```
sudo ctr -n k8s.io container ls
```

Output: 
```
CONTAINER                                                           IMAGE                                                                                                      RUNTIME
..snip
306743d42e5a2e5a4cf754013506d6f3f416ca0a669d89c984d903b4bda5d39f    docker.io/weaveworks/weave-npc:2.8.1                                                                       io.containerd.runc.v2
4599ba8660c7c8efc85499000434271c55536441743f7686dbcea74f247f46e9    k8s.gcr.io/kube-proxy:v1.21.0                                                                              io.containerd.runc.v2
701bf14f65b25d2c4955744837bde03c1afe5f76cec2c985f34fb67eabe50b28    docker.io/library/nginx:1.14.2                                                                             io.containerd.runc.v2
```

We can see that the `nginx` containers were created with `containerd`

Now that this node has been successfully migrated to containerd, migrate the rest of the worker nodes. 
# Congrats! 
