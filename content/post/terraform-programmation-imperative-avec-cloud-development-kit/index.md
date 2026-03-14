---
title: "Terraform : programmation impérative avec Cloud Development Kit"
date: 2020-09-05
tags:
- DevOps
- InfrastructureAsCode
- Scaleway
- Terraform
---

Le logiciel [Terraform](https://www.terraform.io/) est aujourd'hui très utile dès que l'on souhaite faire de l'Infrastructure as Code, notamment par sa capacité à créer, mettre à jour mais aussi supprimer tout une pile applicative chez un fournisseur de services.

A l'image d'autres outils comme [Ansible](https://www.ansible.com/), [Chef](https://www.chef.io/) ou encore [Puppet](https://puppet.com/), Terraform utilise un langage dédié de type déclaratif pour décrire les ressources qui vont être approvisionnées.

Ce langage dédié, le HashiCorp Configuration Language (HCL) dans le cas de Terraform, peut-être vu comme une faiblesse car :

* il impose d'acquérir de nouvelles connaissances spécifiques
* il peut poser problème pour s'intégrer avec la boîte à outils des développeurs
* il n'est surtout pas aussi riche que la plupart des langages de programmation impératifs

Certains projets comme [Pulumi](https://www.pulumi.com/) l'ont bien compris et proposent de gérer une infrastructure avec des langages plus classiques comme JavaScript, Python ou encore Go, tout en conservant l'idempotence des solutions avec langage dédié.

[HashiCorp](https://www.hashicorp.com/), la société à l'origine de Terraform, vient de prendre en compte ce besoin et a annoncé sur son blog un nouveau projet : [Cloud Development Kit (CDK) for Terraform](https://www.hashicorp.com/blog/cdk-for-terraform-enabling-python-and-typescript-support/)

{{< info >}}
A noter que ce projet est basé sur les mêmes composants techniques que AWS utilise pour son [Cloud Development Kit (CDK)](https://aws.amazon.com/cdk/), d'où son nom.
{{< /info >}}

Dans cet article je vais vous présenter une petite infrastructure hébergée chez [Scaleway](https://www.scaleway.com/), décrite d'abord dans le langage dédié de Terraform puis son adaptation en TypeScript à l'aide de CDK for Terraform.

## Utilisation de HCL avec Terraform

Les ressources que l'on va gérer sont les suivantes :

- 2 adresses IP publiques
- 1 groupe de sécurité permettant le ping et le SSH
- 1 source de données correspondant à l'image Ubuntu Bionic, ce qui permettra de récupérer son identifiant 
- 3 serveurs de type ```DEV1-S``` dont seuls les 2 premiers auront une adresse IP publique

Par ailleurs j'afficherais en sortie de l'exécution de Terraform les adresses IP publiques ainsi que les adresses IP privées des serveurs.

Le code source Terraform correspondant à cette infrastructure est par exemple le suivant :

```terraform
provider "scaleway" {
  version = "1.16.0"
}

variable "ip_count" {
  default = 2
}

variable "server_count" {
  default = 3
}

variable "server_type" {
  default = "DEV1-S"
}

variable "server_image" {
  default = "ubuntu_bionic"
}

resource "scaleway_instance_ip" "cdktf" {
  count = var.ip_count
}

resource "scaleway_instance_security_group" "cdktf" {
  name                    = "cdktf"
  stateful                = true
  inbound_default_policy  = "drop"
  outbound_default_policy = "accept"

  inbound_rule {
    action   = "accept"
    protocol = "ICMP"
  }

  inbound_rule {
    action   = "accept"
    protocol = "TCP"
    port     = 22
  }
}

data "scaleway_marketplace_image_beta" "cdktf" {
  label = var.server_image
}

resource "scaleway_instance_server" "cdktf" {
  count             = var.server_count
  type              = var.server_type
  image             = data.scaleway_marketplace_image_beta.cdktf.id
  name              = "cdktf-${count.index + 1}"
  security_group_id = scaleway_instance_security_group.cdktf.id
  ip_id             = count.index < var.ip_count ? scaleway_instance_ip.cdktf[count.index].id : null
}

output "public_ips" {
  value = scaleway_instance_ip.cdktf[*].address
}

output "private_ips" {
  value = scaleway_instance_server.cdktf[*].private_ip
}
```
{{< info >}}
A noter l'utilisation de variables d'environnement pour les secrets permettant la connexion à l'API Scaleway ainsi que pour certaines valeurs par défaut comme la région.
{{< /info >}}

L'application de ce code source se fait avec les commandes ```terraform plan```, ```terraform apply``` et ```terraform destroy``` habituelles. 

## Utilisation de TypeScript avec CDK for Terraform

{{< info >}}
A noter que CDK for Terraform propose également Python comme possible langage.
{{< /info >}}

### Installation de CDK for Terraform

L'utilisation de CDK for Terraform en langage TypeScript nécessite quelques pré-requis :

* [Terraform](https://www.terraform.io/) évidement, dans une version >= 0.12
* [Node.js](https://nodejs.org/) pour interpréter le code TypeScript, en version >= 12.16
* [Yarn](https://yarnpkg.com/) pour gérer les paquets Node.js, en version >= 1.21

La première opération que nous allons lancer est l'installation globale, au niveau système, de l'outillage CDK for Terraform :

```shell
$ sudo npm install -g cdktf-cli
```

### Préparation du projet

Nous allons ensuite initialiser notre projet CDK for Terraform en précisant, pour cet exemple, de ne pas s'interfacer avec le service en ligne Terraform Cloud :

```shell
$ mkdir cdktf-scaleway
$ cd cdktf-scaleway
$ cdktf init --template typescript --local
```

En plus des fichiers habituels pour un logiciel écrit en TypeScript, on retrouve dans l'arborescence du projet :

* un fichier de configuration ```cdktf.json``` pour CDF for Terraform, pour l'instant utilisant le provider Terraform AWS :

```json
{
  "language": "typescript",
  "app": "npm run --silent compile && node main.js",
  "terraformProviders": [
    "aws@~> 2.0"
  ]
}
```

* un fichier d'aide ```help``` rappelant différentes commandes utiles au projet
* un fichier de description ```main.ts``` de notre infrastructure, pour l'instant sans aucune ressources :

```typescript
import { Construct } from 'constructs';
import { App, TerraformStack } from 'cdktf';

class MyStack extends TerraformStack {
  constructor(scope: Construct, name: string) {
    super(scope, name);

    // define resources here

  }
}

const app = new App();
new MyStack(app, 'cdktf-scaleway');
app.synth();
```

### Ajout du provider Scaleway

Comme expliqué plus haut, le projet CDK for Terraform est pré-configuré avec AWS. Nous allons donc ajouter le provider Scaleway dans son fichier de configuration JSON :

```json
{
  "language": "typescript",
  "app": "npm run --silent compile && node main.js",
  "terraformProviders": [
    "aws@~> 2.0",
    "scaleway@~> 1.16"
  ]
}
```

En plus de cette configuration, il faut à présent générer les ```constructs```, un équivalent côté CDK des types de ressource de Terraform :

```shell
$ cdktf get 
Generated typescript constructs in the output directory: .gen
```

Cela a pour effet de créer un fichier TypeScript pour chaque type de ressource du provider Scaleway :

```shell
$ ls .gen/providers/scaleway
account-ssh-key.ts                data-scaleway-instance-security-group.ts  data-scaleway-security-group.ts   instance-security-group.ts  lb-backend-beta.ts      registry-namespace-beta.ts  user-data.ts
baremetal-server.ts               data-scaleway-instance-server.ts          data-scaleway-volume.ts           instance-server.ts          lb-beta.ts              scaleway-provider.ts        volume-attachment.ts
data-scaleway-account-ssh-key.ts  data-scaleway-instance-volume.ts          index.ts                          instance-volume.ts          lb-certificate-beta.ts  security-group-rule.ts      volume.ts
data-scaleway-baremetal-offer.ts  data-scaleway-lb-ip-beta.ts               instance-ip-reverse-dns.ts        ip-reverse-dns.ts           lb-frontend-beta.ts     security-group.ts
data-scaleway-bootscript.ts       data-scaleway-marketplace-image-beta.ts   instance-ip.ts                    ip.ts                       lb-ip-beta.ts           server.ts
data-scaleway-image.ts            data-scaleway-registry-image-beta.ts      instance-placement-group.ts       k8s-cluster-beta.ts         object-bucket.ts        ssh-key.ts
data-scaleway-instance-image.ts   data-scaleway-registry-namespace-beta.ts  instance-security-group-rules.ts  k8s-pool-beta.ts            rdb-instance-beta.ts    token.ts
```

### Ajout du code TypeScript

Tout est prêt maintenant pour déclarer notre infrastructure dans le fichier ```main.ts```, par exemple de cette façon :

```typescript
import { Construct } from 'constructs';
import { App, TerraformOutput, TerraformStack, Token } from 'cdktf';
import { DataScalewayMarketplaceImageBeta, InstanceIp, InstanceServer, InstanceSecurityGroup } from './.gen/providers/scaleway';

class MyStack extends TerraformStack {
  constructor(scope: Construct, name: string) {
    super(scope, name);

    // define resources here
    const ipCount = 2;
    const serverCount = 3;
    const serverType = 'DEV1-S';
    const serverImage = 'ubuntu_bionic';

    const ips: InstanceIp[] = [];

    for (let i = 0; i < ipCount; i++) {
      const ip = new InstanceIp(this, `cdktf-ip-${i}`);

      ips.push(ip);
    }

    const securityGroup = new InstanceSecurityGroup(this, 'cdktf-security-group', {
      name: 'cdktf',
      stateful: true,
      inboundDefaultPolicy: 'drop',
      outboundDefaultPolicy: 'accept',
      inboundRule: [
        {
          action: 'accept',
          protocol: 'ICMP'
        },
        {
          action: 'accept',
          protocol: 'TCP',
          port: 22
        }
      ]
    });

    const image = new DataScalewayMarketplaceImageBeta(this, 'cdktf-image', {
      label: serverImage
    });

    const servers: InstanceServer[] = [];

    for (let i = 0; i < serverCount; i++) {
      const server = new InstanceServer(this, `cdktf-server-${i}`, {
        type: serverType,
        image: Token.asString(image.id),
        name: `cdktf-${i + 1}`,
        securityGroupId: Token.asString(securityGroup.id),
        ipId: i < ipCount ? Token.asString(ips[i].id) : undefined
      });

      servers.push(server);
    }

    new TerraformOutput(this, 'publics-ips', {
      value: ips.map(ip => ip.address)
    });

    new TerraformOutput(this, 'private-ips', {
      value: servers.map(server => server.privateIp)
    });
  }
}

const app = new App();
new MyStack(app, 'cdktf-scaleway');
app.synth();
```

On retrouve à présent l'ensemble des ressources Terraform de notre infrastructure sous forme d'instances de classes TypeScript.

On peut noter comme différences par rapport au langage HCL :

* une unicité des identifiants des ressources au niveau d'une ```TerraformStack``` et non plus une unicité au niveau d'un type de ressource
* un remplacement de la directive ```count``` de HCL par une boucle TypeScript, ici de type ```for```
* une utilisation de la classe ```Token```pour traiter le cas des variables en évaluation retardée, ce qui est le cas des ```id``` des ressources

Pour ce dernier point qui peut être délicat, se référer à la [documentation](https://github.com/hashicorp/terraform-cdk/blob/master/docs/working-with-cdk-for-terraform/tokens.md) pour en savoir plus.

### Gestion de notre infrastructure

La première façon d'appliquer le code source TypeScript est de lancer la commande ```cdktf synth``` afin de de générer dans le répertoire ```cdktf.out``` du code compatible avec les commandes ```terraform``` habituelles.

La seconde façon, celle que je vous conseille, est de passer par les commandes équivalentes avec ```cdktf```.

La commande ```terraform plan``` qui permet d'afficher le plan d'exécution de Terraform devient ainsi la commande ```cdktf diff``` :

```shell
$ cdktf diff
⠴ initializing cdktf-scaleway...
⠧ planning cdktf-scaleway...
⠙ planning cdktf-scaleway...
Stack: cdktf-scaleway
Resources
 + SCALEWAY_INSTANCE_IP cdktfip0            scaleway_instance_ip.cdktfscaleway_cdktfip0_C3F2203D
 + SCALEWAY_INSTANCE_IP cdktfip1            scaleway_instance_ip.cdktfscaleway_cdktfip1_988D7B00
 + SCALEWAY_INSTANCE_SE cdktfsecuritygroup  scaleway_instance_security_group.cdktfscaleway_cdktfsecuritygroup_C024279A
 + SCALEWAY_INSTANCE_SE cdktfserver0        scaleway_instance_server.cdktfscaleway_cdktfserver0_E94A5AA0
 + SCALEWAY_INSTANCE_SE cdktfserver1        scaleway_instance_server.cdktfscaleway_cdktfserver1_BFAA20A7
 + SCALEWAY_INSTANCE_SE cdktfserver2        scaleway_instance_server.cdktfscaleway_cdktfserver2_86C2D83C

Diff: 6 to create, 0 to update, 0 to delete.
```

Pour la commande ```terraform apply``` qui permet le déploiement de notre infrastructure, c'est la commande ```cdktf deploy``` qu'il faudra lancer :

```
$ cdktf deploy
⠸ initializing cdktf-scaleway...
⠴ planning cdktf-scaleway...
⠸ planning cdktf-scaleway...
⠋ Deploying Stack: cdktf-scaleway
Deploying Stack: cdktf-scaleway
Deploying Stack: cdktf-scaleway
Resources
 ✔ SCALEWAY_INSTANCE_IP cdktfip0            scaleway_instance_ip.cdktfscaleway_cdktfip0_C3F2203D
 ✔ SCALEWAY_INSTANCE_IP cdktfip1            scaleway_instance_ip.cdktfscaleway_cdktfip1_988D7B00
 ✔ SCALEWAY_INSTANCE_SE cdktfsecuritygroup  scaleway_instance_security_group.cdktfscaleway_cdktfsecuritygroup_C024279A
 ✔ SCALEWAY_INSTANCE_SE cdktfserver0        scaleway_instance_server.cdktfscaleway_cdktfserver0_E94A5AA0
 ✔ SCALEWAY_INSTANCE_SE cdktfserver1        scaleway_instance_server.cdktfscaleway_cdktfserver1_BFAA20A7
 ✔ SCALEWAY_INSTANCE_SE cdktfserver2        scaleway_instance_server.cdktfscaleway_cdktfserver2_86C2D83C

Summary: 6 created, 0 updated, 0 destroyed.

Output: cdktfscaleway_privateips_71ECA41F = 10.64.162.6710.64.120.1310.68.112.183
        cdktfscaleway_publicsips_8A46FC6B = 51.15.143.5051.158.69.38
```

A noter un défaut de jeunesse dans l'affichage des listes en sortie mais rien de grave, le fichier d'état Terraform est correct.

On peut ensuite se connecter sur la console d'administration Scaleway pour vérifier que les serveurs ont bien été créés :

{{< figure src="scaleway.webp" >}}

Enfin pour détruire notre infrastructure, c'est la commande ```cdktf destroy``` qui remplace la commande ```terraform destroy``` habituelle : 

```
$ cdktf destroy
⠸ initializing cdktf-scaleway...
⠴ planning cdktf-scaleway...
⠸ planning cdktf-scaleway...
⠋ Destroying Stack: cdktf-scaleway
Destroying Stack: cdktf-scaleway
Resources
 ✔ SCALEWAY_INSTANCE_IP cdktfip0            scaleway_instance_ip.cdktfscaleway_cdktfip0_C3F2203D
 ✔ SCALEWAY_INSTANCE_IP cdktfip1            scaleway_instance_ip.cdktfscaleway_cdktfip1_988D7B00
 ✔ SCALEWAY_INSTANCE_SE cdktfsecuritygroup  scaleway_instance_security_group.cdktfscaleway_cdktfsecuritygroup_C024279A
 ✔ SCALEWAY_INSTANCE_SE cdktfserver0        scaleway_instance_server.cdktfscaleway_cdktfserver0_E94A5AA0
 ✔ SCALEWAY_INSTANCE_SE cdktfserver1        scaleway_instance_server.cdktfscaleway_cdktfserver1_BFAA20A7
 ✔ SCALEWAY_INSTANCE_SE cdktfserver2        scaleway_instance_server.cdktfscaleway_cdktfserver2_86C2D83C

Summary: 6 destroyed.
```

## Conclusion

Comme on a pu le voir à travers cet exemple, il est très aisé de passer d'une infrastructure décrite dans le langage HCL de Terraform à une infrastructure décrite en TypeScript à l'aide de CDK for Terraform.

Sachant que n'importe quel provider Terraform existant devrait fonctionner avec l'outil, son périmètre d'utilisation est très large.

Le produit étant tout jeune, il n'est cependant pas encore recommandé pour une utilisation en production mais j'espère que d'ici quelques mois ce sera le cas.

Au final un produit HashiCorp de plus à ajouter dans votre boîte à outils DevOps !

Si vous avez des questions ou des remarques, n’hésitez pas à me laisser un commentaire.
