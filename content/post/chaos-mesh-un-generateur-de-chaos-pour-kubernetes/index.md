---
title: "Chaos Mesh : un générateur de chaos pour Kubernetes"
date: 2020-08-09
tags:
- Chaos
- Kubernetes
---

Dans cet article je vais vous présenter [Chaos Mesh](https://chaos-mesh.org/) un logiciel de Chaos Engineering pour Kubernetes qui a récemment été accepté comme projet par la Cloud Native Computing Foundation.

Pour la petite histoire le concept de Chaos Engineering est un concept introduit par Netflix à l'occasion de sa migration dans le cloud.

Il s'agit d'expérimenter volontairement des pannes aléatoires sur une infrastructure de production afin d'en tester et d'en améliorer sa résilience.

Pour plus d'informations sur le sujet, je vous renvoie notamment à ces articles :

* [Le guide de Chaos Engineering](https://blog.wescale.fr/2019/09/26/le-guide-de-chaos-engineering-part-1/) par Akram Riahi sur le blog de WeScale
* [Chaos Engineering sur des pannes d’infrastructure](https://blog.octo.com/chaos-engineering-sur-des-pannes-dinfrastructure/) par David Shen sur le blog de Octo
* [Chaos engineering avec Adrian Hornsby](https://electro-monkeys.fr/?p=290) sur le podcast de Electro Monkeys

# Installation

Pour procéder à l'installation de Chaos Mesh, je vais utiliser le chart Helm mis à disposition par les développeurs de l'outil.

Je vais tout d'abord créer le namespace `chaos-mesh` dans lequel l'application sera installée :

```shell
$ kubectl create ns chaos-mesh
namespace/chaos-mesh created
```

Je vais ensuite installer les CustomResourceDefinitions gérées par Chaos Mesh, le chart Helm ne déployant pas cette partie de la configuration Kubernetes :

```shell
$ kubectl apply -f https://raw.githubusercontent.com/chaos-mesh/chaos-mesh/chart-0.1.0/manifests/crd.yaml
customresourcedefinition.apiextensions.k8s.io/iochaos.chaos-mesh.org created
customresourcedefinition.apiextensions.k8s.io/kernelchaos.chaos-mesh.org created
customresourcedefinition.apiextensions.k8s.io/networkchaos.chaos-mesh.org created
customresourcedefinition.apiextensions.k8s.io/podchaos.chaos-mesh.org created
customresourcedefinition.apiextensions.k8s.io/stresschaos.chaos-mesh.org created
customresourcedefinition.apiextensions.k8s.io/timechaos.chaos-mesh.org created
```

Je vais enfin utiliser Helm en version 3 pour déployer une instance `chaos-mesh` du logiciel dans le namespace de même nom :

```shell
$ helm repo add chaos-mesh https://charts.chaos-mesh.org/
"chaos-mesh" has been added to your repositories
$ helm install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --set dashboard.create=true --version 0.1.0
NAME: chaos-mesh
LAST DEPLOYED: Tue Aug  4 00:23:39 2020
NAMESPACE: chaos-mesh
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Make sure chaos-mesh components are running
   kubectl get pods --namespace chaos-mesh -l app.kubernetes.io/instance=chaos-mesh
```

Vous pouvez constater que j'ai paramétré le chart Helm pour procéder à l'installation du tableau de bord de Chaos Mesh car celui-ci n'est pas déployé par défaut.

Se référer à la [documentation](https://github.com/chaos-mesh/chaos-mesh/blob/chart-0.1.0/helm/chaos-mesh/README.md) pour l'ensemble des paramètres possibles, notamment pour :

* limiter l'action de Chaos Mesh à certains namespaces
* configurer le container runtime si votre cluster Kubernetes n'utilise pas Docker
* activer la possibilité d'injecter du chaos dans le noyau Linux des machines hôte
* plus classiquement configurer la persistance du tableau de bord à l'aide d'un PersistentVolumeClaim et également son accès à travers un Ingress

Afin de vérifier que tout est opérationnel, nous pouvons récupérer la liste des pods du namespace `chaos-mesh` :

```shell
$ kubectl -n chaos-mesh get pods
NAME                                        READY   STATUS    RESTARTS   AGE
chaos-daemon-fz28l                          1/1     Running   0          2m
chaos-controller-manager-67c5474894-gmgv6   1/1     Running   0          2m
chaos-dashboard-8465957d9-f7rmt             1/1     Running   0          2m
chaos-daemon-9jzxv                          1/1     Running   0          2m
chaos-daemon-sznsb                          1/1     Running   0          2m
chaos-daemon-x2rkr                          1/1     Running   0          2m
```

Nous pouvons aussi utiliser un outil graphique comme [KubeView](https://github.com/benc-uk/kubeview) pour visualiser l'état des pods :

{{< figure src="kubeview.webp" >}}

L'architecture applicative de Chaos Mesh est constituée des composants suivants:

* un Deployment `chaos-controller-manager` : contrôleur qui va prendre en charge et orchestrer les CustomResourceDefinitions spécifiques à Chaos Mesh
* un DaemonSet `chaos-daemon``` : daemon déployé sur toutes les machines du cluster Kubernetes afin de créer du chaos à bas niveau
* un Deployment `chaos-dashboard` : tableau de bord de Chaos Mesh

{{< warning >}}
**Attention** : le contrôleur va communiquer avec le daemon déployé sur chaque machine en se connectant sur le port `31767` de l'IP principale de la machine. Il faut veiller à ce que ce port ne soit pas filtré au niveau réseau.
{{< /warning >}}

# Expérimentations

Dans Chaos Mesh les types d'expérimentation sont mis en œuvre sous forme de CustomResourceDefinitions.

La liste des expérimentations possibles est déjà très riche et couvre un spectre assez large :

| Type           | Périmètre           | Action possible                             |
|----------------|---------------------|---------------------------------------------|
| `PodChaos`     | Les pods            | Tuer des pods                               |
|                |                     | Rendre indisponible des pods                |
|                |                     | Tuer des containers                         |
| `NetworkChaos` | Le réseau           | Partitionner le réseau                      |
|                |                     | Perdre des paquets réseau                   |
|                |                     | Ajouter de la latence réseau                |
|                |                     | Dupliquer des paquets réseau                |
|                |                     | Corrompre des paquets réseau                |
|                |                     | Occuper de la bande passante réseau         |
| `StressChaos`  | La charge           | Charger la mémoire                          |
|                |                     | Charger le CPU                              |
| `TimeChaos`    | L'horloge           | Ajouter du décalage temps                   |
| `IOChaos`      | Les entrées/sorties | Ajouter de la latence d'entrées/sorties     |
|                |                     | Ajouter des erreurs d'entrées/sorties       |
| `KernelChaos`  | Le noyau Linux      | Ajouter des erreurs dans les appels système |

# Exemple d'expérimentation

{{< info >}}
**Attention** : j'ai rencontré des soucis de stabilité avec la CustomResourceDefinition `StressChaos` que je comptais utiliser dans un premier temps comme exemple.
{{< /info >}}

Comme dans la documentation officielle, je vais installer une application qui va lancer des pings sur une IP donnée, en l’occurrence `1.1.1.1`, et je vais y expérimenter du chaos sous forme de latence réseau.

Les ressources Kubernetes, un `Deployment` et un `Service`, correspondant à l'application sont déclarées dans le fichier `web-show.yaml` suivant :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-show
  labels:
    app: web-show
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-show
  template:
    metadata:
      labels:
        app: web-show
    spec:
      nodeSelector:
        kubernetes.io/hostname: anytrack-1
      containers:
        - name: web-show
          image: pingcap/web-show:latest
          imagePullPolicy: Always
          command:
            - /usr/local/bin/web-show
            - --target-ip=1.1.1.1           # IP ciblée par l'application
          ports:
            - name: http
              containerPort: 8081
---
apiVersion: v1
kind: Service
metadata:
  name: web-show
  labels:
    app: web-show
spec:
  selector:
    app: web-show
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8081
```

Je déploie classiquement l'application dans Kubernetes avec la commande :

```shell
$ kubectl apply -f web-show.yaml 
deployment.apps/web-show created
service/web-show created
```

La ressources Kubernetes implémentant l'expérimentation de chaos, un `NetworkChaos`, est paramétrée dans le fichier `network-delay-web-show.yaml` comme ceci :

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-web-show
spec:
  selector:                                 # Sélecteur de pod
    namespaces:                             # Ici un premier sélecteur sur le namespace
      - default          
    labelSelectors:                         # Puis un second sélecteur sur le label du pod
      app: web-show      
  mode: one                                 # Mode, ici un seul pod à la fois
  action: delay                             # Action spécifique, ici de l'ajout de délai
  delay:
    latency: 100ms                          # Latence réseau, ici 100 millisecondes
  scheduler:
    cron: "@every 1m"                       # Fréquence, ici toutes les minutes
  duration: 20s                             # Durée, ici 20 secondes
```

L'application de l'expérimentation de chaos dans Kubernetes s'effectue tout aussi classiquement avec la commande :

```
$ kubectl apply -f network-delay-web-show.yaml 
networkchaos.chaos-mesh.org/network-delay-web-show created
```

Ne reste plus ensuite qu'à se connecter sur le port HTTP `8081` de l'application `web-show`, éventuellement à l'aide d'un `kubectl port-forward svc/web-show 8081`, pour constater l'effet du chaos sur la latence du ping : 

{{< figure src="web-show.webp" >}}

# Tableau de bord

Comme indiqué au moment de l'installation, j'ai activé le déploiement du tableau de bord de Chaos Mesh.

Il est accessible sur le port HTTP `2333`, en s'aidant si nécessaire de la commande `kubectl port-forward -n chaos-mesh svc/chaos-dashboard 2333`, et propose à l'heure actuelle les fonctionalités suivantes :

* formulaire de création d'une expérimentation
* suivi des événements liés à une expérimentation
* mise en pause d'une expérimentation (ajout de l'annotation `experiment.chaos-mesh.org/pause=true` sur la ressource Kubernetes associée)
* suppression d'un expérimentation, à noter que ses données restent archivées

Ici la fiche web de l'expérimentation que nous avons déployée précédemment :

{{< figure src="chaos-dashboard.webp" >}}

# Plugin Grafana

Si vous disposez d'une infrastructure de supervision Prometheus/Grafana dans votre cluster Kubernetes, sachez qu'il existe également un plugin Grafana en cours de développement.

Il n'est pas encore disponible sur le site de Grafana mais il est installable manuellement en lançant la commande suivante dans le container qui héberge Grafana :

```shell
$ grafana-cli --pluginUrl https://github.com/chaos-mesh/chaos-mesh-datasource/archive/v0.1.1.zip plugins install yeya24-chaosmesh-datasource
installing yeya24-chaosmesh-datasource @
from: https://github.com/chaos-mesh/chaos-mesh-datasource/archive/v0.1.1.zip
into: /var/lib/grafana/plugins

✔ Installed yeya24-chaosmesh-datasource successfully 

Restart grafana after installing plugins . <service grafana-server restart>
```

Comme l'indique la dernière ligne du retour de la commande, il faut relancer Grafana pour que le plugin soit pris en compte, par exemple en faisant un `kubectl rollout restart` sur le `Deployment` de Grafana.

Une fois le plugin installé, un nouveau type de source de données Chaos Mesh est disponible.

Il suffit de le configurer avec l'adresse de notre instance du tableau de bord Chaos Mesh : `http://chaos-dashboard.chaos-mesh:2333/`

Les données de Chaos Mesh sont exploitables dans Grafana sous deux formes :

* sous forme de panneaux de type table
* sous forme d'annotations ajoutées aux graphes

{{< info >}}
**Attention** : j'ai dû augmenter la durée de l'expérimentation pour afficher correctement le graphe suivant, en raison d'un soucis de remontée des données dès que la période affichée devenait trop courte.
{{< /info >}}

{{< figure src="grafana.webp" >}}

# Conclusion

Parmi les logiciels libres de Chaos Engineering dont j'ai connaissance sur Kubernetes, et malgré les quelques soucis techniques rencontrés, Chaos Mesh tiens pour moi le haut du pavé de part ses fonctionnalités et de part la diversité des expérimentations qu'il propose.

La [feuille de route](https://github.com/chaos-mesh/chaos-mesh/blob/master/ROADMAP.md) annoncée est d'ailleurs prometteuse :

* amélioration du tableau de bord
* expérimentations de chaos sur la JVM
* expérimentations de chaos sur les connexions HTTP et GRPC
* scénarios pour grouper plusieurs expérimentations
* vérifications d'état pour évaluer la santé des services

Sur ce, bonnes expérimentations et si vous avez des questions ou des remarques, n’hésitez pas à me laisser un commentaire.
