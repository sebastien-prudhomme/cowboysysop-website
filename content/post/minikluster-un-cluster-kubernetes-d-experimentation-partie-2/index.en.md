---
title: "Minikluster: a cluster Kubernetes experimentation (part 2)"
date: 2018-12-29
image: post/minikluster-un-cluster-kubernetes-d-experimentation-partie-2/chuttersnap-255215-unsplash.jpg
bigimg:
- src: chuttersnap-255215-unsplash.jpg
tags:
- Docker
- Kubernetes
- Rancher
---

Following the construction of our Kubernetes Experiment cluster: a small intermission to show you how to easily add a virtual machine to the Kubernetes cluster and how to update the Kubernetes version.

## Preparing a new virtual machine

We will first create a new virtual machine with the help of Docker Machine by providing it with the same features as the existing virtual machine:

* the virtualization solution used: VirtualBox
* the download address of the ISO image of RancherOS (here in version 1.4.1)
* the number of CPUs of the virtual machine: 1
* the size of the virtual machine disk: about 10 GB
* the size of the virtual machine memory: 2 GB
* the name of the virtual machine: minikluster-2

I will spare you for this time the detail of the return of the order:

```shell
$ docker-machine create --driver virtualbox --virtualbox-boot2docker-url https://github.com/rancher/os/releases/download/v1.4.1/rancheros.iso --virtualbox-cpu-count 1 --virtualbox-disk-size 10000 --virtualbox-memory 2048 minikluster-2
```

As before, once the new virtual machine is available, make sure to run a version of Docker compatible with the version of Kubernetes we use:

```shell
$ docker-machine ssh minikluster-2 sudo ros engine switch docker-17.03.2-ce
```

We can now list the virtual machines managed by Docker Machine, this will allow us to retrieve the IP address of our new machine:

```shell
$ docker-machine ls
NAME            ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
minikluster-1   -        virtualbox   Running   tcp://192.168.99.100:2376           v17.03.2-ce
minikluster-2   -        virtualbox   Running   tcp://192.168.99.101:2376           v17.03.2-ce
```

## Adding the virtual machine to the cluster

The first step in expanding our cluster is to edit the `cluster.yml` file previously created in the current directory to add our new virtual machine to the `nodes` list.

Unlike our first machine, we will only worker the `worker` role to the new machine so that it is only responsible for the execution of Kubernetes orchestrated Docker containers and not for the same cluster management (role `controlplane`), nor hosting the internal cluster database (role `etcd`):

```yaml
nodes:
- address: 192.168.99.100
...
- address: 192.168.99.101
  port: "22"
  internal_address: ""
  role:
  - worker
  hostname_override: minikluster-2
  user: docker
  docker_socket: /var/run/docker.sock
  ssh_key: ""
  ssh_key_path: ~/.docker/machine/machines/minikluster-2/id_rsa
  labels: {}
```

The second step is to restart the Kubernetes cluster deployment command with the RKE tool, the latter being responsible for reconciling the expected state of the cluster expressed in the `cluster.yml` file with the actual state of the cluster:

```shell
$ rke up
INFO[0000] Building Kubernetes cluster
INFO[0000] [dialer] Setup tunnel for host [192.168.99.100]
INFO[0000] [dialer] Setup tunnel for host [192.168.99.101]
INFO[0000] [state] Found local kube config file, trying to get state from cluster
INFO[0000] [state] Fetching cluster state from Kubernetes
INFO[0000] [state] Successfully Fetched cluster state to Kubernetes ConfigMap: cluster-state
INFO[0000] [certificates] Getting Cluster certificates from Kubernetes
INFO[0000] [certificates] Successfully fetched Cluster certificates from Kubernetes
INFO[0000] [network] Deploying port listener containers
INFO[0000] [network] Pulling image [rancher/rke-tools:v0.1.13] on host [192.168.99.101]
INFO[0033] [network] Successfully pulled image [rancher/rke-tools:v0.1.13] on host [192.168.99.101]
INFO[0033] [network] Successfully started [rke-worker-port-listener] container on host [192.168.99.101]
INFO[0033] [network] Port listener containers deployed successfully
INFO[0033] [network] Running control plane -> etcd port checks
INFO[0033] [network] Successfully started [rke-port-checker] container on host [192.168.99.100]
INFO[0033] [network] Running control plane -> worker port checks
INFO[0033] [network] Successfully started [rke-port-checker] container on host [192.168.99.100]
INFO[0034] [network] Running workers -> control plane port checks
INFO[0034] [network] Successfully started [rke-port-checker] container on host [192.168.99.101]
INFO[0034] [network] Successfully started [rke-port-checker] container on host [192.168.99.100]
INFO[0034] [network] Checking KubeAPI port Control Plane hosts
INFO[0034] [network] Removing port listener containers
INFO[0034] [remove/rke-worker-port-listener] Successfully removed container on host [192.168.99.101]
INFO[0034] [network] Port listener containers removed successfully
INFO[0035] [reconcile] Reconciling cluster state
INFO[0035] [reconcile] Check etcd hosts to be deleted
INFO[0035] [reconcile] Check etcd hosts to be added
INFO[0035] [reconcile] Rebuilding and updating local kube config
INFO[0035] Successfully Deployed local admin kubeconfig at [./kube_config_cluster.yml]
INFO[0035] [reconcile] host [192.168.99.100] is active master on the cluster
INFO[0035] [reconcile] Reconciled cluster state successfully
INFO[0035] [certificates] Deploying kubernetes certificates to Cluster nodes
INFO[0040] Successfully Deployed local admin kubeconfig at [./kube_config_cluster.yml]
INFO[0040] [certificates] Successfully deployed kubernetes certificates to Cluster nodes
INFO[0040] Pre-pulling kubernetes images
INFO[0040] [pre-deploy] Pulling image [rancher/hyperkube:v1.11.1-rancher1] on host [192.168.99.101]
INFO[0216] [pre-deploy] Successfully pulled image [rancher/hyperkube:v1.11.1-rancher1] on host [192.168.99.101]
INFO[0216] Kubernetes images pulled successfully
INFO[0216] [etcd] Building up etcd plane..
INFO[0217] [etcd] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0217] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0217] [etcd] Successfully started etcd plane..
INFO[0217] [controlplane] Building up Controller Plane..
INFO[0217] [remove/service-sidekick] Successfully removed container on host [192.168.99.100]
INFO[0217] [healthcheck] Start Healthcheck on service [kube-apiserver] on host [192.168.99.100]
INFO[0217] [healthcheck] service [kube-apiserver] on host [192.168.99.100] is healthy
INFO[0217] [controlplane] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0218] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0218] [healthcheck] Start Healthcheck on service [kube-controller-manager] on host [192.168.99.100]
INFO[0218] [healthcheck] service [kube-controller-manager] on host [192.168.99.100] is healthy
INFO[0218] [controlplane] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0218] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0218] [healthcheck] Start Healthcheck on service [kube-scheduler] on host [192.168.99.100]
INFO[0218] [healthcheck] service [kube-scheduler] on host [192.168.99.100] is healthy
INFO[0219] [controlplane] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0219] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0219] [controlplane] Successfully started Controller Plane..
INFO[0219] [authz] Creating rke-job-deployer ServiceAccount
INFO[0219] [authz] rke-job-deployer ServiceAccount created successfully
INFO[0219] [authz] Creating system:node ClusterRoleBinding
INFO[0219] [authz] system:node ClusterRoleBinding created successfully
INFO[0219] [certificates] Save kubernetes certificates as secrets
INFO[0221] [certificates] Successfully saved certificates as kubernetes secret [k8s-certs]
INFO[0221] [state] Saving cluster state to Kubernetes
INFO[0222] [state] Successfully Saved cluster state to Kubernetes ConfigMap: cluster-state
INFO[0222] [worker] Building up Worker Plane..
INFO[0222] [remove/service-sidekick] Successfully removed container on host [192.168.99.100]
INFO[0222] [healthcheck] Start Healthcheck on service [kubelet] on host [192.168.99.100]
INFO[0222] [healthcheck] service [kubelet] on host [192.168.99.100] is healthy
INFO[0222] [worker] Successfully started [nginx-proxy] container on host [192.168.99.101]
INFO[0222] [worker] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0222] [worker] Successfully started [rke-log-linker] container on host [192.168.99.101]
INFO[0222] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0222] [healthcheck] Start Healthcheck on service [kube-proxy] on host [192.168.99.100]
INFO[0222] [remove/rke-log-linker] Successfully removed container on host [192.168.99.101]
INFO[0222] [healthcheck] service [kube-proxy] on host [192.168.99.100] is healthy
INFO[0222] [worker] Successfully started [kubelet] container on host [192.168.99.101]
INFO[0222] [healthcheck] Start Healthcheck on service [kubelet] on host [192.168.99.101]
INFO[0223] [worker] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0223] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0228] [healthcheck] service [kubelet] on host [192.168.99.101] is healthy
INFO[0228] [worker] Successfully started [rke-log-linker] container on host [192.168.99.101]
INFO[0228] [remove/rke-log-linker] Successfully removed container on host [192.168.99.101]
INFO[0228] [worker] Successfully started [kube-proxy] container on host [192.168.99.101]
INFO[0228] [healthcheck] Start Healthcheck on service [kube-proxy] on host [192.168.99.101]
INFO[0233] [healthcheck] service [kube-proxy] on host [192.168.99.101] is healthy
INFO[0233] [worker] Successfully started [rke-log-linker] container on host [192.168.99.101]
INFO[0234] [remove/rke-log-linker] Successfully removed container on host [192.168.99.101]
INFO[0234] [worker] Successfully started Worker Plane..
INFO[0234] [cleanup] Successfully started [rke-log-cleaner] container on host [192.168.99.101]
INFO[0234] [remove/rke-log-cleaner] Successfully removed container on host [192.168.99.101]
INFO[0234] [sync] Syncing nodes Labels and Taints
INFO[0234] [sync] Successfully synced nodes Labels and Taints
INFO[0234] [network] Setting up network plugin: canal
INFO[0234] [addons] Saving addon ConfigMap to Kubernetes
INFO[0234] [addons] Successfully Saved addon to Kubernetes ConfigMap: rke-network-plugin
INFO[0234] [addons] Executing deploy job..
INFO[0234] [addons] Setting up KubeDNS
INFO[0234] [addons] Saving addon ConfigMap to Kubernetes
INFO[0234] [addons] Successfully Saved addon to Kubernetes ConfigMap: rke-kubedns-addon
INFO[0234] [addons] Executing deploy job..
INFO[0234] [addons] KubeDNS deployed successfully..
INFO[0234] [addons] Setting up Metrics Server
INFO[0234] [addons] Saving addon ConfigMap to Kubernetes
INFO[0234] [addons] Successfully Saved addon to Kubernetes ConfigMap: rke-metrics-addon
INFO[0234] [addons] Executing deploy job..
INFO[0234] [addons] KubeDNS deployed successfully..
INFO[0234] [ingress] Setting up nginx ingress controller
INFO[0234] [addons] Saving addon ConfigMap to Kubernetes
INFO[0234] [addons] Successfully Saved addon to Kubernetes ConfigMap: rke-ingress-controller
INFO[0234] [addons] Executing deploy job..
INFO[0234] [ingress] ingress controller nginx is successfully deployed
INFO[0234] [addons] Setting up user addons
INFO[0234] [addons] no user addons defined
INFO[0234] Finished building Kubernetes cluster successfully
```

We can now retrieve the list of nodes to verify that the new machine has been integrated into the Kubernetes cluster:

```shell
$ kubectl --kubeconfig=kube_config_cluster.yml get nodes
NAME            STATUS    ROLES                      AGE       VERSION
minikluster-1   Ready     controlplane,etcd,worker   27d       v1.11.1
minikluster-2   Ready     worker                     4m        v1.11.1
```

We can also see that the list of pods in the `kube-system` namespace now contains a new pod created by the daemonset `canal` and which is essential for the operation of the Kubernetes network layer on our new virtual machine:

```shell
$ kubectl --kubeconfig kube_config_cluster.yml --namespace kube-system get pods
NAME                                      READY     STATUS      RESTARTS   AGE
canal-95lrl                               3/3       Running     0          4m
canal-q6j9j                               3/3       Running     6          27d
kube-dns-7588d5b5f5-mjs9c                 3/3       Running     6          27d
kube-dns-autoscaler-5db9bbb766-bhzbf      1/1       Running     2          27d
metrics-server-97bc649d5-ng6mq            1/1       Running     2          27d
rke-ingress-controller-deploy-job-wp9cg   0/1       Completed   0          27d
rke-kubedns-addon-deploy-job-dl984        0/1       Completed   0          27d
rke-metrics-addon-deploy-job-fcjxx        0/1       Completed   0          27d
rke-network-plugin-deploy-job-nwgz9       0/1       Completed   0          27d
```

## Upgrading the cluster

Now that we have expanded our cluster, we will just as easily upgrade the Kubernetes software.

The first thing to know is that each version of the RKE tool offers the installation of different versions of Kubernetes, one of which is installed by default.

For version 0.1.9 of RKE, the version of Kubernetes by default is version 1.11.1 as we can for example see when we retrieve the list of nodes of the cluster.

We will now get RKE version 0.1.14: https://github.com/rancher/rke/releases/tag/v0.1.14

The following command allows us to display the version of Kubernetes installed by default as well as the so-called system images that RKE will use to build the cluster:

```shell
$ rke config --system-images
INFO[0000] Generating images list for version [v1.11.5-rancher1-1]:
rancher/coreos-etcd:v3.2.18
rancher/rke-tools:v0.1.15
rancher/k8s-dns-kube-dns-amd64:1.14.10
rancher/k8s-dns-dnsmasq-nanny-amd64:1.14.10
rancher/k8s-dns-sidecar-amd64:1.14.10
rancher/cluster-proportional-autoscaler-amd64:1.0.0
rancher/hyperkube:v1.11.5-rancher1
rancher/coreos-flannel:v0.10.0
rancher/coreos-flannel-cni:v0.3.0
rancher/calico-node:v3.1.3
rancher/calico-cni:v3.1.3
rancher/calico-ctl:v2.0.0
weaveworks/weave-kube:2.1.2
weaveworks/weave-npc:2.1.2
rancher/pause-amd64:3.1
rancher/nginx-ingress-controller:0.16.2-rancher1
rancher/nginx-ingress-controller-defaultbackend:1.4
rancher/metrics-server-amd64:v0.2.1
```

For version 0.1.14 of RKE, the version of Kubernetes by default is version 1.11.5.

If you want to display all versions of Kubernetes recognized by our version of RKE, we can do it with the following command:

```shell
$ rke config --system-images --all >/dev/null
INFO[0000] Generating images list for version [v1.8.11-rancher2-1]:
INFO[0000] Generating images list for version [v1.10.5-rancher1-1]:
INFO[0000] Generating images list for version [v1.12.0-rancher1-1]:
INFO[0000] Generating images list for version [v1.8.10-rancher1-1]:
INFO[0000] Generating images list for version [v1.9.7-rancher2-2]:
INFO[0000] Generating images list for version [v1.10.1-rancher2-1]:
INFO[0000] Generating images list for version [v1.11.2-rancher1-1]:
INFO[0000] Generating images list for version [v1.11.5-rancher1-1]:
INFO[0000] Generating images list for version [v1.12.1-rancher1-1]:
INFO[0000] Generating images list for version [v1.9.5-rancher1-1]:
INFO[0000] Generating images list for version [v1.10.1-rancher1]:
INFO[0000] Generating images list for version [v1.10.5-rancher1-2]:
INFO[0000] Generating images list for version [v1.11.1-rancher1-1]:
INFO[0000] Generating images list for version [v1.11.2-rancher1-2]:
INFO[0000] Generating images list for version [v1.11.3-rancher1-1]:
INFO[0000] Generating images list for version [v1.12.3-rancher1-1]:
INFO[0000] Generating images list for version [v1.9.7-rancher1]:
INFO[0000] Generating images list for version [v1.9.7-rancher2-1]:
INFO[0000] Generating images list for version [v1.10.0-rancher1-1]:
INFO[0000] Generating images list for version [v1.10.3-rancher2-1]:
INFO[0000] Generating images list for version [v1.10.11-rancher1-1]:
INFO[0000] Generating images list for version [v1.8.11-rancher1]:
```

{{% info %}}
I have deliberately redirected the standard output to `/dev/null` order to simplify the return of the command by not displaying the so-called system images.
{{% /info %}}

Even if we are not going to do it for our cluster, it is possible to configure the `cluster.yml` file to use another version of Kubernetes than the one proposed by default: https://rancher.com/docs/rke/v0.1.x/en/config-options/#kubernetes-version

On the other hand, we will modify this `cluster.yml` file to no more impose images called system because it would prevent their climbs of version towards those of our new version of Kubernetes:

```yaml
...
system_images:
  etcd: ""
  alpine: ""
  nginx_proxy: ""
  cert_downloader: ""
  kubernetes_services_sidecar: ""
  kubedns: ""
  dnsmasq: ""
  kubedns_sidecar: ""
  kubedns_autoscaler: ""
  kubernetes: ""
  flannel: ""
  flannel_cni: ""
  calico_node: ""
  calico_cni: ""
  calico_controllers: ""
  calico_ctl: ""
  canal_node: ""
  canal_cni: ""
  canal_flannel: ""
  wave_node: ""
  weave_cni: ""
  pod_infra_container: ""
  ingress: ""
  ingress_backend: ""
  metrics_server: ""
...
```

As before, we will launch the Kubernetes cluster deployment command with the RKE tool:

```shell
$ rke up
INFO[0000] Building Kubernetes cluster
INFO[0000] [dialer] Setup tunnel for host [192.168.99.100]
INFO[0000] [dialer] Setup tunnel for host [192.168.99.101]
INFO[0000] [state] Found local kube config file, trying to get state from cluster
INFO[0000] [state] Fetching cluster state from Kubernetes
INFO[0000] [state] Successfully Fetched cluster state to Kubernetes ConfigMap: cluster-state
INFO[0000] [certificates] Getting Cluster certificates from Kubernetes
INFO[0000] [certificates] Successfully fetched Cluster certificates from Kubernetes
INFO[0000] [network] No hosts added existing cluster, skipping port check
INFO[0000] [reconcile] Reconciling cluster state
INFO[0000] [reconcile] Check etcd hosts to be deleted
INFO[0000] [reconcile] Check etcd hosts to be added
INFO[0000] [reconcile] Rebuilding and updating local kube config
INFO[0000] Successfully Deployed local admin kubeconfig at [./kube_config_cluster.yml]
INFO[0000] [reconcile] host [192.168.99.100] is active master on the cluster
INFO[0000] [reconcile] Reconciled cluster state successfully
INFO[0000] [certificates] Deploying kubernetes certificates to Cluster nodes
INFO[0000] [certificates] Pulling image [rancher/rke-tools:v0.1.15] on host [192.168.99.100]
INFO[0000] [certificates] Pulling image [rancher/rke-tools:v0.1.15] on host [192.168.99.101]
INFO[0061] [certificates] Successfully pulled image [rancher/rke-tools:v0.1.15] on host [192.168.99.101]
INFO[0068] [certificates] Successfully pulled image [rancher/rke-tools:v0.1.15] on host [192.168.99.100]
INFO[0074] Successfully Deployed local admin kubeconfig at [./kube_config_cluster.yml]
INFO[0074] [certificates] Successfully deployed kubernetes certificates to Cluster nodes
INFO[0074] Pre-pulling kubernetes images
INFO[0074] [pre-deploy] Pulling image [rancher/hyperkube:v1.11.5-rancher1] on host [192.168.99.100]
INFO[0074] [pre-deploy] Pulling image [rancher/hyperkube:v1.11.5-rancher1] on host [192.168.99.101]
INFO[0410] [pre-deploy] Successfully pulled image [rancher/hyperkube:v1.11.5-rancher1] on host [192.168.99.101]
INFO[0411] [pre-deploy] Successfully pulled image [rancher/hyperkube:v1.11.5-rancher1] on host [192.168.99.100]
INFO[0411] Kubernetes images pulled successfully
INFO[0411] [etcd] Building up etcd plane..
INFO[0421] [etcd] Successfully updated [etcd] container on host [192.168.99.100]
INFO[0425] [etcd] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0426] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0426] [etcd] Successfully started etcd plane..
INFO[0426] [controlplane] Building up Controller Plane..
INFO[0426] [remove/service-sidekick] Successfully removed container on host [192.168.99.100]
INFO[0441] [controlplane] Successfully updated [kube-apiserver] container on host [192.168.99.100]
INFO[0442] [healthcheck] Start Healthcheck on service [kube-apiserver] on host [192.168.99.100]
INFO[0456] [healthcheck] service [kube-apiserver] on host [192.168.99.100] is healthy
INFO[0458] [controlplane] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0460] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0468] [controlplane] Successfully updated [kube-controller-manager] container on host [192.168.99.100]
INFO[0468] [healthcheck] Start Healthcheck on service [kube-controller-manager] on host [192.168.99.100]
INFO[0468] [healthcheck] service [kube-controller-manager] on host [192.168.99.100] is healthy
INFO[0471] [controlplane] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0473] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0481] [controlplane] Successfully updated [kube-scheduler] container on host [192.168.99.100]
INFO[0481] [healthcheck] Start Healthcheck on service [kube-scheduler] on host [192.168.99.100]
INFO[0481] [healthcheck] service [kube-scheduler] on host [192.168.99.100] is healthy
INFO[0484] [controlplane] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0486] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0486] [controlplane] Successfully started Controller Plane..
INFO[0486] [authz] Creating rke-job-deployer ServiceAccount
INFO[0486] [authz] rke-job-deployer ServiceAccount created successfully
INFO[0486] [authz] Creating system:node ClusterRoleBinding
INFO[0486] [authz] system:node ClusterRoleBinding created successfully
INFO[0486] [certificates] Save kubernetes certificates as secrets
INFO[0488] [certificates] Successfully saved certificates as kubernetes secret [k8s-certs]
INFO[0488] [state] Saving cluster state to Kubernetes
INFO[0488] [state] Successfully Saved cluster state to Kubernetes ConfigMap: cluster-state
INFO[0488] [state] Saving cluster state to cluster nodes
INFO[0491] [state] Successfully started [cluster-state-deployer] container on host [192.168.99.100]
INFO[0493] [remove/cluster-state-deployer] Successfully removed container on host [192.168.99.100]
INFO[0497] [state] Successfully started [cluster-state-deployer] container on host [192.168.99.101]
INFO[0498] [remove/cluster-state-deployer] Successfully removed container on host [192.168.99.101]
INFO[0498] [worker] Building up Worker Plane..
INFO[0498] [remove/service-sidekick] Successfully removed container on host [192.168.99.100]
INFO[0504] [worker] Successfully updated [kubelet] container on host [192.168.99.100]
INFO[0505] [healthcheck] Start Healthcheck on service [kubelet] on host [192.168.99.100]
INFO[0508] [worker] Successfully updated [nginx-proxy] container on host [192.168.99.101]
INFO[0511] [healthcheck] service [kubelet] on host [192.168.99.100] is healthy
INFO[0513] [worker] Successfully started [rke-log-linker] container on host [192.168.99.101]
INFO[0515] [remove/rke-log-linker] Successfully removed container on host [192.168.99.101]
INFO[0516] [remove/service-sidekick] Successfully removed container on host [192.168.99.101]
INFO[0518] [worker] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0523] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0523] [worker] Successfully updated [kubelet] container on host [192.168.99.101]
INFO[0523] [healthcheck] Start Healthcheck on service [kubelet] on host [192.168.99.101]
INFO[0529] [healthcheck] service [kubelet] on host [192.168.99.101] is healthy
INFO[0533] [worker] Successfully started [rke-log-linker] container on host [192.168.99.101]
INFO[0535] [remove/rke-log-linker] Successfully removed container on host [192.168.99.101]
INFO[0535] [worker] Successfully updated [kube-proxy] container on host [192.168.99.100]
INFO[0536] [healthcheck] Start Healthcheck on service [kube-proxy] on host [192.168.99.100]
INFO[0541] [healthcheck] service [kube-proxy] on host [192.168.99.100] is healthy
INFO[0545] [worker] Successfully updated [kube-proxy] container on host [192.168.99.101]
INFO[0545] [healthcheck] Start Healthcheck on service [kube-proxy] on host [192.168.99.101]
INFO[0545] [healthcheck] service [kube-proxy] on host [192.168.99.101] is healthy
INFO[0548] [worker] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0549] [worker] Successfully started [rke-log-linker] container on host [192.168.99.101]
INFO[0551] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0551] [remove/rke-log-linker] Successfully removed container on host [192.168.99.101]
INFO[0551] [worker] Successfully started Worker Plane..
INFO[0551] [sync] Syncing nodes Labels and Taints
INFO[0551] [sync] Successfully synced nodes Labels and Taints
INFO[0551] [network] Setting up network plugin: canal
INFO[0551] [addons] Saving addon ConfigMap to Kubernetes
INFO[0551] [addons] Successfully Saved addon to Kubernetes ConfigMap: rke-network-plugin
INFO[0551] [addons] Executing deploy job..
INFO[0577] [addons] Setting up KubeDNS
INFO[0577] [addons] Saving addon ConfigMap to Kubernetes
INFO[0577] [addons] Successfully Saved addon to Kubernetes ConfigMap: rke-kubedns-addon
INFO[0577] [addons] Executing deploy job..
INFO[0577] [addons] KubeDNS deployed successfully..
INFO[0577] [addons] Setting up Metrics Server
INFO[0577] [addons] Saving addon ConfigMap to Kubernetes
INFO[0577] [addons] Successfully Saved addon to Kubernetes ConfigMap: rke-metrics-addon
INFO[0577] [addons] Executing deploy job..
INFO[0577] [addons] KubeDNS deployed successfully..
INFO[0577] [ingress] Setting up nginx ingress controller
INFO[0577] [addons] Saving addon ConfigMap to Kubernetes
INFO[0577] [addons] Successfully Saved addon to Kubernetes ConfigMap: rke-ingress-controller
INFO[0577] [addons] Executing deploy job..
INFO[0604] [ingress] ingress controller nginx is successfully deployed
INFO[0604] [addons] Setting up user addons
INFO[0604] [addons] no user addons defined
INFO[0604] Finished building Kubernetes cluster successfully
```

We can finally confirm the transition to version 1.11.5 of Kubernetes by displaying the list of nodes of the cluster:

```shell
$ kubectl --kubeconfig=kube_config_cluster.yml get nodes
NAME            STATUS    ROLES                      AGE       VERSION
minikluster-1   Ready     controlplane,etcd,worker   28d       v1.11.5
minikluster-2   Ready     worker                     1d        v1.11.5
```

This is the end of this intermission dedicated to the modification of our cluster. Next time we'll see how to use the Terraform tool to automate everything we've seen so far.

If you have questions or comments, do not hesitate to leave me a comment.

{{% info %}}
This post was originally written in French and then translated into English with the help of [Google Translate](https://translate.google.com/).
{{% /info %}}
