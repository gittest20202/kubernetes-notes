## Migrate Kubernetes from Docker to Containerd
**Kubernetes decided to deprecate Docker as container runtime after v1.20**

**Explanation of why Kubernetes decided to deprecate Docker can be found here: **[Reason Of Depricate Docker](https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/)

**In this post, we are going to migrate Kubernetes cluster from Docker to Containerd. These changes will apply to all nodes in the cluster. I recommend starting migrating from worker nodes.**

##Prepare node
**First of all, scheduling must be disabled and all unnecessary workloads, except daemon sets, must be evicted.

- Start by cordoning the node
```
$ kubectl cordon k8s-worker-3
node/k8s-worker-3 cordoned
```

- Drain node
```
$ kubectl drain k8s-worker-3 --ignore-daemonsets --delete-emptydir-data
node/k8s-worker-3 already cordoned
node/k8s-worker-3 evicted
```

- Stop Kubelet and docker
```
$ systemctl stop kubelet
$ systemctl stop docker
```

##Switch to containerd
**Node is ready to be migrated to containerd. Start by removing docker as it will not be needed anymore**
```
$ apt purge docker-ce docker-ce-cli
```

**Ensure containerd is installed**
```
$ ctr -n moby container list
```

**Ensure that config file for containerd in /etc/containerd/config.toml is present. You can generate it with**
```
$ mkdir -p /etc/containerd
$ containerd config default | sudo tee /etc/containerd/config.toml
```

**If you are using the systemd as a cgroup driver, you must configure it in containerd config. In /etc/containerd/config.toml Add**
```
<...>
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  <...>
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true # <--- This line
<...>
```

**Restart containerd**
```
$ systemctl restart containerd
```

**Edit /var/lib/kubelet/kubeadm-flags.env file by adding container runtime flags**

* --container-runtime=remote
* --container-runtime-endpoint=unix:///run/containerd/containerd.sock

**kubeadm-flags.env file should now look something like this**
```
$ cat /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.2 --container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
```

**Start Kubelet**
```
$ systemctl start kubelet
```

**Check cluster status**
```
$ kubectl get nodes
```
**Uncordon the node if everything looks good**
```
$ kubectl uncordon k8s-worker-3
```

**Repeat the procedure for all nodes (one by one)**

###Post-migration
**Congratulations on migrating your cluster to containerd. However, few things left to do to be fully migrated. Let's free up some space by removing docker-related folders. They will not be needed anymore**
```
$ rm -r /etc/docker
$ rm -r /var/lib/docker
$ rm -r /var/lib/dockershim
```

**If you are using kubeadm to manage your cluster initialization, joins, and updates, you might want to re-annotate nodes, so kubeadm will not get confused on your next update whether you are using docker or containterd as a runtime**

```
$ kubectl annotate node k8s-worker-3 --overwrite kubeadm.alpha.kubernetes.io/cri-socket=unix:///run/containerd/containerd.sock
```

**Note that removals and annotate must be executed on all nodes after the whole cluster will be migrated to containerd and ensured that the migration was successful.**
