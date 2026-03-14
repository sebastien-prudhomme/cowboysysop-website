---
title: "Automatiser la configuration d'Umami avec Terraform"
date: 2026-03-06
tags:
- DevOps
- InfrastructureAsCode
- Terraform
- Umami
---

[Umami](https://umami.is/) est une excellente alternative open-source et respectueuse de la vie privée à Google Analytics.

Lorsque l'on déploie Umami, on souhaite peut-être gérer sa configuration, comme la déclaration des sites web à suivre, via une approche Infrastructure as Code.

Le problème est qu'à ce jour, il n'existe pas de provider Terraform officiel pour Umami.

Faut-il pour autant repasser par des clics dans l'interface web ou des scripts maison ? Pas du tout !

Dans cet article, je vais vous montrer comment contourner cette limitation en utilisant des providers Terraform génériques pour interagir directement avec l'API Umami.

## Les providers nécessaires

Puisque nous n'avons pas de provider dédié, nous allons nous appuyer sur deux providers, l'un officiel, l'autre communautaire, pour effectuer des appels HTTP et manipuler des API RESTful :

```terraform
terraform {
  required_providers {
    http = {
      source  = "hashicorp/http"
      version = "3.5.0"
    }

    restful = {
      source  = "magodo/restful"
      version = "0.25.1"
    }
  }
}
```

* Le provider `http` va nous servir à l'authentification initiale.
* Le provider `restful` est un provider très puissant qui permet de créer, lire, mettre à jour et supprimer des ressources sur n'importe quelle API respectant les standards RESTful.

## Authentification à l'API Umami

L'API Umami requiert l'utilisation d'un token d'authentification pour chaque requête. Avant de pouvoir créer nos sites web, nous devons donc récupérer ce token.

Nous allons d'abord déclarer les variables nécessaires à la connexion :

```terraform
variable "umami_url" {
  type        = string
  description = "L'URL de base de l'instance Umami (ex: https://umami.example.com)"
}

variable "umami_username" {
  type        = string
  description = "Le nom d'utilisateur pour se connecter à l'API Umami"
}

variable "umami_password" {
  type        = string
  description = "Le mot de passe associé à l'utilisateur Umami"
  sensitive   = true
}
```

Ensuite, nous utilisons la source de données `http` pour effectuer un appel sur [l'endpoint d'authentification](https://umami.is/docs/api/authentication) de l'API :

```terraform
data "http" "umami_login" {
  url    = "${var.umami_url}/api/auth/login"
  method = "POST"

  request_body = jsonencode({
    username = var.umami_username
    password = var.umami_password
  })
}

locals {
  umami_token = jsondecode(data.http.umami_login.response_body).token
}
```

La réponse au format JSON de cet appel HTTP contient le fameux token.

Nous utilisons une variable locale `umami_token` associée à la fonction `jsondecode` pour l'extraire et le rendre facilement utilisable par la suite.

## Configuration du provider RESTful

Maintenant que nous avons notre token, nous pouvons configurer le provider `restful` :

```terraform
provider "restful" {
  alias    = "umami"
  base_url = "${var.umami_url}/api"

  security = {
    http = {
      token = {
        token = local.umami_token
      }
    }
  }
}
```

Vous remarquerez l'utilisation de la directive `alias = "umami"`. Pourquoi utiliser un alias ici ?

Le provider `restful` est par nature très générique. Il est tout à fait possible que vous ayez besoin de l'utiliser pour interagir avec plusieurs API différentes au sein du même projet Terraform.

L'alias permet de créer une instance spécifique du provider, configurée exclusivement pour une utilisation avec l'API Umami.

Cela permet de référencer cette configuration lors de la déclaration de nos ressources, évitant ainsi tout conflit ou effet de bord indésirable.

## Gérer les sites web dans Umami

Avec notre provider prêt à l'emploi, nous pouvons maintenant déclarer un site web.

La documentation de l'API Umami concernant les [sites web](https://umami.is/docs/api/websites) nous indique comment structurer notre requête :

```terraform
resource "restful_resource" "www_example_com" {
  provider      = restful.umami
  path          = "/websites"
  read_path     = "$(path)/$(body.id)"
  update_method = "POST"

  body = {
    name   = "My website"
    domain = "www.example.com"
  }
}
```

Détaillons les paramètres de cette ressource :

* `provider = restful.umami` : nous appelons explicitement l'instance du provider que nous avons configurée avec l'alias
* `path = "/websites"` : c'est l'endpoint de l'API pour gérer les sites web
* `read_path = "$(path)/$(body.id)"` : définit comment Terraform doit aller lire l'état de la ressource, ici on concatène le chemin avec l'ID retourné dans le corps de la réponse lors de la création de cette même ressource
* `update_method = "POST"` : l'API Umami utilise de manière non standard la méthode POST pour les mises à jour, d'où la nécessité de forcer ce paramètre
* `body` : le contenu JSON envoyé, décrivant les caractéristiques de notre site web

## Importer des ressources existantes

Si vous avez déjà créé des sites web à la main dans votre instance Umami et que vous souhaitez les rapatrier sous Terraform, le provider `restful` gère parfaitement l'importation de l'existant.

Voici comment construire le bloc `import` :

```terraform
import {
  provider = restful.umami
  to       = restful_resource.www_example_com
  id = jsonencode({
    id   = "/websites/<id>"
    path = "/websites"
    body = {
      name   = null
      domain = null
    }
  })
}
```

L'identifiant d'importation attendu par ce provider `restful` est particulier : c'est un objet JSON qui renseigne le chemin exact de la ressource, le chemin de base et enfin le squelette de l'objet attendu.

## Conclusion

Grâce à la flexibilité de Terraform et de ses providers génériques, l'absence de provider officiel n'est pas un obstacle bloquant.

Le provider `restful` en est un parfait exemple et permet d'intégrer assez facilement l'API Umami dans vos pipelines de déploiement.

Si vous avez des questions ou des remarques sur cette approche, n’hésitez pas à me laisser un commentaire.
