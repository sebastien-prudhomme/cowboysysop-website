---
title: "Minikluster : un cluster Kubernetes d’expérimentation (partie 1)"
date: 2018-10-27
image: /post/minikluster-un-cluster-kubernetes-d-experimentation-partie-1/banner.jpg
bigimg:
- src: /post/minikluster-un-cluster-kubernetes-d-experimentation-partie-1/banner.jpg
tags:
- Docker
- Kubernetes
- Rancher
---

Dans cette série d’articles nous allons installer un cluster Kubernetes local à des fins d’expérimentation.

Il existe certes des solutions comme [Minikube](https://kubernetes.io/docs/setup/minikube/) ou [Minishift](https://www.okd.io/minishift/) qui permettent déjà d’installer facilement Kubernetes sur un poste local mais aucune à ma connaissance ne propose d‘installer un cluster à plusieurs nœuds comme nous allons le faire.

Dans un premier temps nous allons effectuer une installation de Kubernetes à l’aide de la ligne de commande, sans plus d’automatisation que cela. Nous nous aiderons pour cela de l’outil Docker Machine, de la distribution Linux RancherOS et enfin de l’outil Rancher Kubernetes Engine (RKE).

Dans un second temps nous verrons comment automatiser cette installation à l’aide de l’outil Terraform, avec pour objectif de créer, modifier et détruire le cluster Kubernetes le plus simplement possible en suivant l’approche Infrastructure As Code.

# Pré-requis

Afin de procéder à l’installation de notre cluster Kubernetes, les pré-requis suivants sont demandés :

* un poste client sous Linux avec si possible 16 Go de mémoire (ne rien espérer d’utilisable en dessous de 8 Go)
* la solution de virtualisation [VirtualBox](https://www.virtualbox.org/) installée et fonctionnelle sur le poste Linux (je ne rentrerais pas dans les détails, vous trouverez tout ce qu’il faut sur le site officiel)
* une connaissance de base des containers Docker et de l’orchestrateur Kubernetes

Les logiciels utilisés par la suite étant multi-plateformes, une installation sur un autre type d’OS (Mac OS, Windows) ou sur un autre hyperviseur (KVM, VMware Fusion) devrait être possible en adaptant la procédure.

# Kubernetes

![](kubernetes.png)

Le seul élément de Kubernetes nécessaire sur le poste client est l’outil en ligne de commande Kubectl.

Pour son installation, je vous suggère de passer par cette méthode de la documentation officielle : https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-binary-using-curl

{{< warning >}}
**Attention** : veillez à sélectionner la version 1.11.1 pour ne pas avoir de mauvaise surprise dans la suite de cet article.
{{< /warning >}}

Une fois installé, vous pouvez le lancer pour vérifier qu’il fonctionne et vous rendre compte des différentes commandes qu’il propose :

```shell
$ kubectl
kubectl controls the Kubernetes cluster manager.

Find more information at: https://kubernetes.io/docs/reference/kubectl/overview/

Basic Commands (Beginner):
  create         Create a resource from a file or from stdin.
  expose         Take a replication controller, service, deployment or pod and expose it as a new Kubernetes Service
  run            Run a particular image on the cluster
  set            Set specific features on objects

Basic Commands (Intermediate):
  explain        Documentation of resources
  get            Display one or many resources
  edit           Edit a resource on the server
  delete         Delete resources by filenames, stdin, resources and names, or by resources and label selector

Deploy Commands:
  rollout        Manage the rollout of a resource
  scale          Set a new size for a Deployment, ReplicaSet, Replication Controller, or Job
  autoscale      Auto-scale a Deployment, ReplicaSet, or ReplicationController

Cluster Management Commands:
  certificate    Modify certificate resources.
  cluster-info   Display cluster info
  top            Display Resource (CPU/Memory/Storage) usage.
  cordon         Mark node as unschedulable
  uncordon       Mark node as schedulable
  drain          Drain node in preparation for maintenance
  taint          Update the taints on one or more nodes

Troubleshooting and Debugging Commands:
  describe       Show details of a specific resource or group of resources
  logs           Print the logs for a container in a pod
  attach         Attach to a running container
  exec           Execute a command in a container
  port-forward   Forward one or more local ports to a pod
  proxy          Run a proxy to the Kubernetes API server
  cp             Copy files and directories to and from containers.
  auth           Inspect authorization

Advanced Commands:
  apply          Apply a configuration to a resource by filename or stdin
  patch          Update field(s) of a resource using strategic merge patch
  replace        Replace a resource by filename or stdin
  wait           Experimental: Wait for one condition on one or many resources
  convert        Convert config files between different API versions

Settings Commands:
  label          Update the labels on a resource
  annotate       Update the annotations on a resource
  completion     Output shell completion code for the specified shell (bash or zsh)

Other Commands:
  alpha          Commands for features in alpha
  api-resources  Print the supported API resources on the server
  api-versions   Print the supported API versions on the server, in the form of "group/version"
  config         Modify kubeconfig files
  plugin         Runs a command-line plugin
  version        Print the client and server version information

Usage:
  kubectl [flags] [options]

Use "kubectl <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).
```

# Docker Machine

![](docker-machine.png)

[Docker Machine](https://docs.docker.com/machine/) est un outil qui va nous permettre de créer facilement les machines virtuelles qui serviront à l’hébergement de Kubernetes et de les équiper du Docker Engine.

Il est compatible avec un grand nombre de fournisseurs de virtualisation cloud ou privés : Amazon Web Services, Google Compute Engine, Microsoft Azure, VMware vSphere et bien entendu VirtualBox.

Pour son installation, je vous renvoie à la documentation officielle : https://docs.docker.com/machine/install-machine/

{{< warning >}}
**Attention** : veillez à sélectionner la version 0.15.0 pour ne pas avoir de mauvaise surprise dans la suite de cet article.
{{< /warning >}}

Comme pour Kubectl, vous pouvez vérifier son fonctionnement en le lançant sans option particulière :

```shell
$ docker-machine
Usage: docker-machine [OPTIONS] COMMAND [arg...]

Create and manage machines running Docker.

Version: 0.15.0, build b48dc28d

Author:
  Docker Machine Contributors - <https://github.com/docker/machine>

Options:
  --debug, -D                                           Enable debug mode
  --storage-path, -s "/home/sebastien/.docker/machine"  Configures storage path [$MACHINE_STORAGE_PATH]
  --tls-ca-cert                                         CA to verify remotes against [$MACHINE_TLS_CA_CERT]
  --tls-ca-key                                          Private key to generate certificates [$MACHINE_TLS_CA_KEY]
  --tls-client-cert                                     Client cert to use for TLS [$MACHINE_TLS_CLIENT_CERT]
  --tls-client-key                                      Private key used in client TLS auth [$MACHINE_TLS_CLIENT_KEY]
  --github-api-token                                    Token to use for requests to the Github API [$MACHINE_GITHUB_API_TOKEN]
  --native-ssh                                          Use the native (Go-based) SSH implementation. [$MACHINE_NATIVE_SSH]
  --bugsnag-api-token                                   BugSnag API token for crash reporting [$MACHINE_BUGSNAG_API_TOKEN]
  --help, -h                                            show help
  --version, -v                                         print the version

Commands:
  active                Print which machine is active
  config                Print the connection config for machine
  create                Create a machine
  env                   Display the commands to set up the environment for the Docker client
  inspect               Inspect information about a machine
  ip                    Get the IP address of a machine
  kill                  Kill a machine
  ls                    List machines
  provision             Re-provision existing machines
  regenerate-certs      Regenerate TLS Certificates for a machine
  restart               Restart a machine
  rm                    Remove a machine
  ssh                   Log into or run a command on a machine with SSH.
  scp                   Copy files between machines
  mount                 Mount or unmount a directory from a machine with SSHFS.
  start                 Start a machine
  status                Get the status of a machine
  stop                  Stop a machine
  upgrade               Upgrade a machine to the latest version of Docker
  url                   Get the URL of a machine
  version               Show the Docker Machine version or a machine docker version
  help                  Shows a list of commands or help for one command

Run 'docker-machine COMMAND --help' for more information on a command.
```

# RancherOS

![](rancheros.png)

[RancherOS](https://rancher.com/rancher-os/) est une distribution Linux légère spécialement conçue pour l’hébergement de containers Docker.

Contrairement à l’image Boot2docker installée par défaut par Docker Machine, RancherOS peut aussi servir de socle système pour un environnement de production.

RancherOS est par exemple compatible avec les mécanismes de configuration au boot de type cloud-init ainsi qu’avec le mécanisme guestinfo de VMware vSphere.

RancherOS possède aussi l’avantage de pouvoir basculer facilement d’une version de Docker à une autre, ce que nous allons faire pour rester dans les versions recommandées pour faire fonctionner Kubernetes.

Pour notre cluster Kubernetes d’expérimentation, nous allons commencer par créer une première machine virtuelle avec l’aide de Docker Machine en lui fournissant :

* la solution de virtualisation utilisée : VirtualBox
* l’adresse de téléchargement de l’image ISO de RancherOS (ici dans sa version 1.4.1)
* le nombre de CPU de la machine virtuelle : 1
* la taille du disque de la machine virtuelle : environ 10 Go
* la taille de la mémoire de la machine virtuelle : 2 Go
* le nom de la machine virtuelle : minikluster-1

```shell
$ docker-machine create --driver virtualbox --virtualbox-boot2docker-url https://github.com/rancher/os/releases/download/v1.4.1/rancheros.iso --virtualbox-cpu-count 1 --virtualbox-disk-size 10000 --virtualbox-memory 2048 minikluster-1
Creating CA: /home/sebastien/.docker/machine/certs/ca.pem
Creating client certificate: /home/sebastien/.docker/machine/certs/cert.pem
Running pre-create checks...
(minikluster-1) Image cache directory does not exist, creating it at /home/sebastien/.docker/machine/cache...
Creating machine...
(minikluster-1) Downloading /home/sebastien/.docker/machine/cache/boot2docker.iso from https://github.com/rancher/os/releases/download/v1.4.1/rancheros.iso...
(minikluster-1) 0%....10%....20%....30%....40%....50%....60%....70%....80%....90%....100%
(minikluster-1) Creating VirtualBox VM...
(minikluster-1) Creating SSH key...
(minikluster-1) Starting the VM...
(minikluster-1) Check network to re-create if needed...
(minikluster-1) Waiting for an IP...
Waiting for machine to be running, this may take a few minutes...
Detecting operating system of created instance...
Waiting for SSH to be available...
Detecting the provisioner...
Provisioning with rancheros...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
Checking connection to Docker...
Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: docker-machine env minikluster-1
```

Nous pouvons ensuite vérifier la version de Docker installée par défaut sous RancherOS en listant les machines virtuelles gérées par Docker Machine :

```shell
$ docker-machine ls
NAME            ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
minikluster-1   -        virtualbox   Running   tcp://192.168.99.100:2376           v18.03.1-ce
```

La version de Docker 18.03.1-ce n’étant pas garantie pour fonctionner avec la version de Kubernetes 1.11 que nous allons déployer, nous allons la remplacer à l’aide de l’outil de configuration de RancherOS par la version compatible la plus récente, à savoir la 17.03.2-ce :

```shell
$ docker-machine ssh minikluster-1 sudo ros engine switch docker-17.03.2-ce
time="2018-09-30T21:12:59Z" level=info msg="Project [os]: Starting project "
time="2018-09-30T21:12:59Z" level=error msg="Missing the image: Error: No such image: docker.io/rancher/os-docker:17.03.2"
time="2018-09-30T21:12:59Z" level=error msg="Missing the image: Error: No such image: docker.io/rancher/os-docker:17.03.2"
time="2018-09-30T21:12:59Z" level=info msg="[0/18] [docker]: Starting "
Pulling docker (docker.io/rancher/os-docker:17.03.2)...
17.03.2: Pulling from rancher/os-docker
8d6b05d859b8: Pulling fs layer
8d6b05d859b8: Verifying Checksum
8d6b05d859b8: Download complete
8d6b05d859b8: Pull complete
Digest: sha256:ef61048c4719cb3f4c61ac9566235420db613695594185617f38ea6a31f5ca85
Status: Downloaded newer image for rancher/os-docker:17.03.2
time="2018-09-30T21:13:22Z" level=info msg="Recreating docker"
time="2018-09-30T21:13:22Z" level=info msg="[1/18] [docker]: Started "
```

Nous rajouterons d’autres machines virtuelle plus tard, afin de montrer comment l’outil utilisé pour l’installation de Kubernetes peut agrandir un cluster déjà existant sans aucune difficulté.

# Rancher Kubernetes Engine (RKE)

![](rke.png)

Rancher Kubernetes Engine, que nous appellerons RKE par la suite, est l’outil qui va nous permettre d’installer un cluster Kubernetes sur la machine virtuelle que nous venons de créer.

Il possède plusieurs atouts non négligeables par rapport à d’autres méthodes d’installation de Kubernetes :

* simple à installer : un seul binaire
* simple à configurer : un seul fichier YAML
* simple à utiliser : une seule commande pour non seulement lancer le déploiement du cluster Kubernetes mais aussi pour le mettre à jour

Pour son installation, comme pour Docker Machine, je vous renvoie à la documentation officielle : https://rancher.com/docs/rke/v0.1.x/en/installation/#download-the-rke-binary

{{< warning >}}
**Attention** : veillez à télécharger la version 0.1.9 pour rester compatible avec cet article.
{{< /warning >}}

Comme pour les outils précédemment installés, vous pouvez ensuite exécuter le binaire pour vérifier son fonctionnement et ses commandes :

```shell
$ rke
NAME:
   rke - Rancher Kubernetes Engine, an extremely simple, lightning fast Kubernetes installer that works everywhere

USAGE:
   rke [global options] command [command options] [arguments...]

VERSION:
   v0.1.9

AUTHOR(S):
   Rancher Labs, Inc.

COMMANDS:
     up       Bring the cluster up
     remove   Teardown the cluster and clean cluster nodes
     version  Show cluster Kubernetes version
     config   Setup cluster configuration
     etcd     etcd snapshot save/restore operations in k8s cluster
     help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --debug, -d    Debug logging
   --help, -h     show help
   --version, -v  print the version
```

Une fois RKE installé, il faut créer le fichier YAML qui va servir à sa configuration.

Afin d’initialiser ce fichier YAML, nous allons nous servir de la commande de configuration présente dans le binaire RKE, en lui fournissant les informations suivantes et en laissant les autres paramètres à leur valeur par défaut :

* adresse SSH de l’hôte : 192.168.99.100 (à adapter en fonction de ce que vous retourne Docker Machine dans sa liste de machines virtuelles)
* chemin vers la clé privée SSH de l’hôte : ~/.docker/machine/machines/minikluster-1/id_rsa (créée automatiquement par Docker Machine lors de l’installation de la machine virtuelle)
* utilisateur SSH de l’hôte : docker (créé automatiquement lors de l’installation de RancherOS)
* activation du rôle worker (exécution des containers Docker orchestrés par Kubernetes) : oui
* activation du rôle etcd (base de données interne de Kubernetes) : oui
* surcharge du nom d’hôte : minikluster-1

```shell
$ rke config
[+] Cluster Level SSH Private Key Path [~/.ssh/id_rsa]:
[+] Number of Hosts [1]:
[+] SSH Address of host (1) [none]: 192.168.99.100
[+] SSH Port of host (1) [22]:
[+] SSH Private Key Path of host (192.168.99.100) [none]: ~/.docker/machine/machines/minikluster-1/id_rsa
[+] SSH User of host (192.168.99.100) [ubuntu]: docker
[+] Is host (192.168.99.100) a Control Plane host (y/n)? [y]:
[+] Is host (192.168.99.100) a Worker host (y/n)? [n]: y
[+] Is host (192.168.99.100) an etcd host (y/n)? [n]: y
[+] Override Hostname of host (192.168.99.100) [none]: minikluster-1
[+] Internal IP of host (192.168.99.100) [none]:
[+] Docker socket path on host (192.168.99.100) [/var/run/docker.sock]:
[+] Network Plugin Type (flannel, calico, weave, canal) [canal]:
[+] Authentication Strategy [x509]:
[+] Authorization Mode (rbac, none) [rbac]:
[+] Kubernetes Docker image [rancher/hyperkube:v1.11.1-rancher1]:
[+] Cluster domain [cluster.local]:
[+] Service Cluster IP Range [10.43.0.0/16]:
[+] Enable PodSecurityPolicy [n]:
[+] Cluster Network CIDR [10.42.0.0/16]:
[+] Cluster DNS Service IP [10.43.0.10]:
[+] Add addon manifest URLs or YAML files [no]:
```

Le résultat de cette commande est sous forme d’un nouveau fichier nommé `cluster.yml` dans le répertoire courant, fichier qui contient notamment les informations que nous venons de renseigner :

```yaml
nodes:
- address: 192.168.99.100
  port: "22"
  internal_address: ""
  role:
  - controlplane
  - worker
  - etcd
  hostname_override: minikluster-1
  user: docker
  docker_socket: /var/run/docker.sock
  ssh_key: ""
  ssh_key_path: ~/.docker/machine/machines/minikluster-1/id_rsa
  labels: {}
```

Si vous souhaitez en savoir plus sur ce qu’il est possible de configurer dans ce fichier YAML, je vous renvoie comme toujours à la documentation officielle : https://rancher.com/docs/rke/v0.1.x/en/config-options/

Ne reste plus à présent qu’à lancer le déploiement de Kubernetes sur la machine virtuelle avec l’outil RKE :

```shell
$ rke up
INFO[0000] Building Kubernetes cluster
INFO[0000] [dialer] Setup tunnel for host [192.168.99.100]
INFO[0000] [network] Deploying port listener containers
INFO[0000] [network] Pulling image [rancher/rke-tools:v0.1.13] on host [192.168.99.100]
INFO[0031] [network] Successfully pulled image [rancher/rke-tools:v0.1.13] on host [192.168.99.100]
INFO[0032] [network] Successfully started [rke-etcd-port-listener] container on host [192.168.99.100]
INFO[0033] [network] Successfully started [rke-cp-port-listener] container on host [192.168.99.100]
INFO[0033] [network] Successfully started [rke-worker-port-listener] container on host [192.168.99.100]
INFO[0033] [network] Port listener containers deployed successfully
INFO[0033] [network] Running control plane -> etcd port checks
INFO[0033] [network] Successfully started [rke-port-checker] container on host [192.168.99.100]
INFO[0033] [network] Running control plane -> worker port checks
INFO[0033] [network] Successfully started [rke-port-checker] container on host [192.168.99.100]
INFO[0033] [network] Running workers -> control plane port checks
INFO[0034] [network] Successfully started [rke-port-checker] container on host [192.168.99.100]
INFO[0034] [network] Checking KubeAPI port Control Plane hosts
INFO[0034] [network] Removing port listener containers
INFO[0034] [remove/rke-etcd-port-listener] Successfully removed container on host [192.168.99.100]
INFO[0034] [remove/rke-cp-port-listener] Successfully removed container on host [192.168.99.100]
INFO[0034] [remove/rke-worker-port-listener] Successfully removed container on host [192.168.99.100]
INFO[0034] [network] Port listener containers removed successfully
INFO[0034] [certificates] Attempting to recover certificates from backup on [etcd,controlPlane] hosts
INFO[0034] [certificates] Successfully started [cert-fetcher] container on host [192.168.99.100]
INFO[0034] [certificates] No Certificate backup found on [etcd,controlPlane] hosts
INFO[0034] [certificates] Generating CA kubernetes certificates
INFO[0034] [certificates] Generating Kubernetes API server certificates
INFO[0034] [certificates] Generating Kube Controller certificates
INFO[0034] [certificates] Generating Kube Scheduler certificates
INFO[0035] [certificates] Generating Kube Proxy certificates
INFO[0035] [certificates] Generating Node certificate
INFO[0035] [certificates] Generating admin certificates and kubeconfig
INFO[0035] [certificates] Generating etcd-192.168.99.100 certificate and key
INFO[0035] [certificates] Generating Kubernetes API server aggregation layer requestheader client CA certificates
INFO[0035] [certificates] Generating Kubernetes API server proxy client certificates
INFO[0036] [certificates] Temporarily saving certs to [etcd,controlPlane] hosts
INFO[0041] [certificates] Saved certs to [etcd,controlPlane] hosts
INFO[0041] [reconcile] Reconciling cluster state
INFO[0041] [reconcile] This is newly generated cluster
INFO[0041] [certificates] Deploying kubernetes certificates to Cluster nodes
INFO[0047] Successfully Deployed local admin kubeconfig at [./kube_config_cluster.yml]
INFO[0047] [certificates] Successfully deployed kubernetes certificates to Cluster nodes
INFO[0047] Pre-pulling kubernetes images
INFO[0047] [pre-deploy] Pulling image [rancher/hyperkube:v1.11.1-rancher1] on host [192.168.99.100]
INFO[0229] [pre-deploy] Successfully pulled image [rancher/hyperkube:v1.11.1-rancher1] on host [192.168.99.100]
INFO[0229] Kubernetes images pulled successfully
INFO[0230] [etcd] Building up etcd plane..
INFO[0230] [etcd] Pulling image [rancher/coreos-etcd:v3.2.18] on host [192.168.99.100]
INFO[0243] [etcd] Successfully pulled image [rancher/coreos-etcd:v3.2.18] on host [192.168.99.100]
INFO[0244] [etcd] Successfully started [etcd] container on host [192.168.99.100]
INFO[0244] [etcd] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0244] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0244] [etcd] Successfully started etcd plane..
INFO[0244] [controlplane] Building up Controller Plane..
INFO[0245] [controlplane] Successfully started [kube-apiserver] container on host [192.168.99.100]
INFO[0245] [healthcheck] Start Healthcheck on service [kube-apiserver] on host [192.168.99.100]
INFO[0258] [healthcheck] service [kube-apiserver] on host [192.168.99.100] is healthy
INFO[0258] [controlplane] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0258] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0258] [controlplane] Successfully started [kube-controller-manager] container on host [192.168.99.100]
INFO[0258] [healthcheck] Start Healthcheck on service [kube-controller-manager] on host [192.168.99.100]
INFO[0264] [healthcheck] service [kube-controller-manager] on host [192.168.99.100] is healthy
INFO[0264] [controlplane] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0264] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0264] [controlplane] Successfully started [kube-scheduler] container on host [192.168.99.100]
INFO[0264] [healthcheck] Start Healthcheck on service [kube-scheduler] on host [192.168.99.100]
INFO[0269] [healthcheck] service [kube-scheduler] on host [192.168.99.100] is healthy
INFO[0269] [controlplane] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0269] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0269] [controlplane] Successfully started Controller Plane..
INFO[0269] [authz] Creating rke-job-deployer ServiceAccount
INFO[0269] [authz] rke-job-deployer ServiceAccount created successfully
INFO[0269] [authz] Creating system:node ClusterRoleBinding
INFO[0269] [authz] system:node ClusterRoleBinding created successfully
INFO[0269] [certificates] Save kubernetes certificates as secrets
INFO[0269] [certificates] Successfully saved certificates as kubernetes secret [k8s-certs]
INFO[0269] [state] Saving cluster state to Kubernetes
INFO[0270] [state] Successfully Saved cluster state to Kubernetes ConfigMap: cluster-state
INFO[0270] [worker] Building up Worker Plane..
INFO[0270] [remove/service-sidekick] Successfully removed container on host [192.168.99.100]
INFO[0270] [worker] Successfully started [kubelet] container on host [192.168.99.100]
INFO[0270] [healthcheck] Start Healthcheck on service [kubelet] on host [192.168.99.100]
INFO[0276] [healthcheck] service [kubelet] on host [192.168.99.100] is healthy
INFO[0276] [worker] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0276] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0276] [worker] Successfully started [kube-proxy] container on host [192.168.99.100]
INFO[0276] [healthcheck] Start Healthcheck on service [kube-proxy] on host [192.168.99.100]
INFO[0281] [healthcheck] service [kube-proxy] on host [192.168.99.100] is healthy
INFO[0282] [worker] Successfully started [rke-log-linker] container on host [192.168.99.100]
INFO[0282] [remove/rke-log-linker] Successfully removed container on host [192.168.99.100]
INFO[0282] [worker] Successfully started Worker Plane..
INFO[0282] [sync] Syncing nodes Labels and Taints
INFO[0282] [sync] Successfully synced nodes Labels and Taints
INFO[0282] [network] Setting up network plugin: canal
INFO[0282] [addons] Saving addon ConfigMap to Kubernetes
INFO[0282] [addons] Successfully Saved addon to Kubernetes ConfigMap: rke-network-plugin
INFO[0282] [addons] Executing deploy job..
INFO[0292] [addons] Setting up KubeDNS
INFO[0292] [addons] Saving addon ConfigMap to Kubernetes
INFO[0292] [addons] Successfully Saved addon to Kubernetes ConfigMap: rke-kubedns-addon
INFO[0292] [addons] Executing deploy job..
INFO[0297] [addons] KubeDNS deployed successfully..
INFO[0297] [addons] Setting up Metrics Server
INFO[0297] [addons] Saving addon ConfigMap to Kubernetes
INFO[0297] [addons] Successfully Saved addon to Kubernetes ConfigMap: rke-metrics-addon
INFO[0297] [addons] Executing deploy job..
INFO[0302] [addons] KubeDNS deployed successfully..
INFO[0302] [ingress] Setting up nginx ingress controller
INFO[0302] [addons] Saving addon ConfigMap to Kubernetes
INFO[0303] [addons] Successfully Saved addon to Kubernetes ConfigMap: rke-ingress-controller
INFO[0303] [addons] Executing deploy job..
INFO[0308] [ingress] ingress controller nginx is successfully deployed
INFO[0308] [addons] Setting up user addons
INFO[0308] [addons] no user addons defined
INFO[0308] Finished building Kubernetes cluster successfully
```

La commande va par ailleurs créer dans le répertoire courant un fichier de configuration `kube_config_cluster.yml` contenant les informations nécessaires pour se connecter sur le cluster Kubernetes que l’on vient de déployer.

Afin de vérifier que tout est opérationnel, nous pouvons par exemple récupérer la liste des nœuds qui participent au fonctionnement du cluster Kubernetes :

```shell
$ kubectl --kubeconfig=kube_config_cluster.yml get nodes
NAME            STATUS    ROLES                      AGE       VERSION
minikluster-1   Ready     controlplane,etcd,worker   4m        v1.11.1
```

Nous pouvons également récupérer la liste des pods du namespace `kube-system` :

```shell
$ kubectl --kubeconfig kube_config_cluster.yml --namespace kube-system get pods
NAME                                      READY     STATUS      RESTARTS   AGE
canal-95m29                               3/3       Running     0          4m
kube-dns-7588d5b5f5-wgrhz                 3/3       Running     0          4m
kube-dns-autoscaler-5db9bbb766-rlx6l      1/1       Running     0          4m
metrics-server-97bc649d5-xgjrr            1/1       Running     0          4m
rke-ingress-controller-deploy-job-g4mng   0/1       Completed   0          4m
rke-kubedns-addon-deploy-job-rxqs7        0/1       Completed   0          4m
rke-metrics-addon-deploy-job-2fscd        0/1       Completed   0          4m
rke-network-plugin-deploy-job-bkrnw       0/1       Completed   0          4m
```

Voila, notre cluster Kubernetes est à présent opérationnel, pour l’instant sur une seule machine virtuelle, et ce sera tout pour cette première partie.

Si vous avez des questions ou des remarques, n’hésitez pas à me laisser un commentaire.
