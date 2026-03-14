---
title: "Automating Umami configuration with Terraform"
url: /en/post/automating-umami-configuration-with-terraform
date: 2026-03-06
tags:
- DevOps
- InfrastructureAsCode
- Terraform
- Umami
---

[Umami](https://umami.is/) is an excellent open-source, privacy-friendly alternative to Google Analytics.

When deploying Umami, you might want to manage its configuration, such as declaring the websites to track—using an Infrastructure as Code approach.

The problem is that, to date, there is no official Terraform provider for Umami.

Does this mean we have to revert to clicking through the web interface or writing custom scripts? Not at all!

In this article, I will show you how to work around this limitation by using generic Terraform providers to interact directly with the Umami API.

## Required providers

Since we don't have a dedicated provider, we will rely on two providers, one official, one community-supported, to make HTTP calls and interact with RESTful APIs:

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

* The `http` provider will be used for initial authentication.
* The `restful` provider is a very powerful provider that allows creating, reading, updating, and deleting resources on any API that complies with RESTful standards.

## Authenticating with the Umami API

The Umami API requires the use of an authentication token for each request. Before we can create our websites, we need to retrieve this token.

First, we will declare the variables needed for the connection:

```terraform
variable "umami_url" {
  type        = string
  description = "The base URL of the Umami instance (e.g., https://umami.example.com)"
}

variable "umami_username" {
  type        = string
  description = "The username to connect to the Umami API"
}

variable "umami_password" {
  type        = string
  description = "The password associated with the Umami user"
  sensitive   = true
}
```

Next, we use the `http` data source to make a call to the API's [authentication endpoint](https://umami.is/docs/api/authentication):

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

The JSON response of this HTTP call contains the token.

We use a local variable `umami_token` paired with the `jsondecode` function to extract it and make it easily usable later.

## Configuring the RESTful provider

Now that we have our token, we can configure the `restful` provider:

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

You'll notice the use of the `alias = "umami"` directive. Why use an alias here?

The `restful` provider is generic by nature. It's quite possible that you'll need to use it to interact with several different APIs within the same Terraform project.

The alias allows creating a specific instance of the provider, configured exclusively for use with the Umami API.

This makes it possible to reference this configuration when declaring our resources, thereby avoiding any conflicts or unwanted side effects.

## Managing websites in Umami

With our provider ready to go, we can now declare a website.

The Umami API documentation regarding [websites](https://umami.is/docs/api/websites) tells us how to structure our request:

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

Let's break down the parameters of this resource:

* `provider = restful.umami`: we explicitly call the provider instance we configured with the alias
* `path = "/websites"`: this is the API endpoint to manage websites
* `read_path = "$(path)/$(body.id)"`: defines how Terraform should read the state of the resource, here we concatenate the path with the ID returned in the response body when creating this same resource
* `update_method = "POST"`: the Umami API uses the POST method for updates in a non-standard way, hence the need to force this parameter
* `body`: the JSON content sent, describing the characteristics of our website

## Importing existing resources

If you have already created websites manually in your Umami instance and want to bring them under Terraform management, the `restful` provider handles importing existing resources perfectly.

Here is how to build the `import` block:

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

The import identifier expected by this `restful` provider is quite specific: it's a JSON object providing the exact resource path, the base path, and finally the skeleton of the expected object.

## Conclusion

Thanks to the flexibility of Terraform and its generic providers, the lack of an official provider is not a blocking issue.

The `restful` provider is a perfect example of this and makes it fairly easy to integrate the Umami API into your deployment pipelines.

If you have any questions or remarks about this approach, feel free to leave me a comment.

{{< info >}}
This post was originally written in French and then translated into English with the help of [Google Gemini](https://gemini.google.com/).
{{< /info >}}
