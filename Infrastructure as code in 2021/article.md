There are numerous advantages to using IaC, here are few examples:

- **Automation** Scripted deployments reduce surface for human error and are faster to execute
- **Reproducibility** Being able to recreate the same environment every time or multiple identical environment based on the same templates
- **Security** Permission to production environments can be granted only to deployment service accounts, reducing risks
- **Costs** Being able to destroy and recreate environments quickly can enable fast de-provisioning of expensive resources when they are not used

# What are the options?

This article is **Azure-centric**, but most of what will be discussed will also apply to **AWS**, **GCP** and other clouds and targets. Cloud-specific systems such as **ARM templates/Bicep** may be comparable with **AWS CloudFormation**, **GCP Cloud Deployment Manager** and to **K8s YAML** for on-premises targets.

**The code examples below all deploy exactly the same resources**

## ARM templates

Azure-native way of scripting resources; the language used is JSON.

I have worked extensively with **ARM templates**, but I still get an headache when I open one for the first time.

**JSON** *is* human readable, but not intuitive.

It requires a great deal of boilerplate and, **JSON** being **JSON**, naturally requires an abundance of curly braces, double quotes and other symbols that represent a substantial distraction from the actual meaningful content and hence easily lead to [cognitive overload](https://www.teachingenglish.org.uk/article/cognitive-overload). Many people I spoke to dislike it for this precise reason.

The nesting doesn't help and it handles modules poorly.

Enough being negative, what's great about it? It's Azure's mother-tongue. It supports all Azure resources as soon as they are available.

In addition, if you use the Azure Resource Explorer, you will find exactly what has been deployed in your subscription and that will be in **JSON** and compatible with your **ARM templates**;

If you create something in the portal, you can easily export its current configuration as an **ARM template**.

This is a template that creates a **resource group** and a **storage account**:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "rgName": {
      "type": "string",
      "defaultValue": "rg-arm"
    },
    "rgLocation": {
      "type": "string",
      "defaultValue": "West Europe"
    }
  },
  "variables": {},
  "resources": [
    {
      "type": "Microsoft.Resources/resourceGroups",
      "apiVersion": "2018-05-01",
      "location": "[parameters('rgLocation')]",
      "name": "[parameters('rgName')]",
      "properties": {}
    },
    {
      "type": "Microsoft.Resources/deployments",
      "apiVersion": "2017-05-10",
      "name": "storageDeployment",
      "resourceGroup": "[parameters('rgName')]",
      "dependsOn": [
        "[resourceId('Microsoft.Resources/resourceGroups/', parameters('rgName'))]"
      ],
      "properties": {
        "mode": "Incremental",
        "template": {
          "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
          "contentVersion": "1.0.0.0",
          "parameters": {},
          "variables": {},
          "resources": [
            {
              "type": "Microsoft.Storage/storageAccounts",
              "apiVersion": "2017-10-01",
              "name": "[concat('sa', uniquestring(subscription().id))]",
              "location": "West Europe",
              "kind": "StorageV2",
              "sku": {
                "name": "Standard_LRS"
              },
              "properties": {
                "supportsHttpsTrafficOnly": true
              }
            }
          ],
          "outputs": {}
        }
      }
    }
  ],
  "outputs": {}
}
```

### Pros

- Native deployment history tracking
- Always up to date with new resources
- Supported by Microsoft

### Cons

- Unfriendly language
- Hard to manage modules
- Complexity increases exponentially for large environments

## Bicep

OK, now take **ARM templates**, remove all the downsides and here you have **Bicep**.

A **Bicep template** is pretty much the same as an **ARM template**, in fact it transpiles to **ARM JSON** to use the same underlying deployment system, but **Bicep** addresses the two main cons of **ARM**.

The first massive advantage over **ARM** is the language; it's a bespoke DSL, easy to write and understand.

Modularisation is also easier than **ARM** as it allows to reference other templates in the file system.

Being the same as **ARM** means also that is Azure-only, which, currently, is the main drawback.

Same resources from the **ARM template** above, but in **Bicep**:

```bicep
targetScope = 'subscription'

resource rg 'Microsoft.Resources/resourceGroups@2020-01-01' = {
  name: 'rg-bicep'
  location: 'West Europe'
  scope: subscription()
}

module stgModule './storageAccount.bicep' = {
  name: 'storageDeploy'
  scope: rg
  params: {
    location: rg.location
  }
}
```

```bicep
param location string

resource stg 'Microsoft.Storage/storageAccounts@2019-06-01' = {
  name: 'sa${uniqueString(resourceGroup().id)}'
  location: location
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
  }  
  sku: {
    name: 'Standard_LRS'
  }
}
```

Even without syntax highlighting from the blog engine, this immediately looks awesome, way more readable and succinct. In my opinion a great option if you are Azure-only and want to avoid the burden of state management. You will miss out on clean-up of resources that **Pulumi** and **Terraform** achieve with an external state, but it is worth evaluating.

### Pros

- Same pros as ARM
- Expressive (concise) language

### Cons

- Azure-only
- No resources clean-up

## Terraform

Terraform is the exceptional attempt at a multi-cloud, human-readable, infrastructure as code tool; a successful attempt; it is quickly becoming the industry standard for cloud deployments.

The **HashiCorp** tool uses an external state storage, this registers all the resources deployed and enables destruction of the resources removed from the templates and clean up of an entire environment.

Uses a custom DSL called **HCL**, quite friendly and understandable by anyone at a glance.

One major downside is that is a limited language; it covers the basic conditionals, loops, variables (poorly in my opinion).

It's a beautiful solution for simple deployments, but it gets pretty frustrating when attempting to be more clever with the logic.

Same resources as above in **Terraform** this time. I am using **Azure** blob as a state storage, but it has several options including the local file system.

One big downside if that often you find yourself in a situation where a new resource or a new feature of a resource comes up in Azure (and I presume the same for other providers), but you have to wait for the **Terraform** team to implement it to leverage it in idiomatic **Terraform**. You can always deploy **ARM templates** from within **Terraform**, but it is ugly and you miss a richer diff experience.

Modules support in **Terraform** is also great and it allows you to reference modules directly from external **Git** repositories.

```terraform
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-iac-demo"
    storage_account_name = "saiacdemo"
    container_name       = "terraform"
    key                  = "demo.tfstate"
  }
}

variable "subscription_id" {
  type      = string
  sensitive = true
}

variable "client_id" {
  type      = string
  sensitive = true
}

variable "client_secret" {
  type      = string
  sensitive = true
}

variable "tenant_id" {
  type      = string
  sensitive = true
}

provider "azurerm" {
  features {}

  subscription_id = var.subscription_id
  client_id       = var.client_id
  client_secret   = var.client_secret
  tenant_id       = var.tenant_id
}

resource "random_id" "storage_account" {
  byte_length = 8
}

resource "azurerm_resource_group" "example" {
  name     = "rg-terraform"
  location = "West Europe"
}

resource "azurerm_storage_account" "example" {
  name                      = "sa${lower(random_id.storage_account.hex)}"
  resource_group_name       = azurerm_resource_group.example.name
  location                  = azurerm_resource_group.example.location
  account_tier              = "Standard"
  account_replication_type  = "LRS"
  enable_https_traffic_only = true
}
```

### Pros

- Unofficial industry standard as of 2021
- Language and CLI are incredibly easy to understand and to use

### Cons

- **HCL** has its limits and simple logic can turn into complex templates
- Storage of the state is in plain text, including secrets; responsibility for securing it lies with the user
- Tooling and code completion is not always great and misses "compile"-time checks
- Being an open-source tool, is supported by the community only unless you pay for **Terraform Enterprise**
- Resources support is delayed and sometimes quite heavily

## Pulumi

[Dulcis in fundo...](https://en.wiktionary.org/wiki/dulcis_in_fundo) my favourite **IaC** tool as of today.

The people at **Pulumi** had a great intuition:

Using general purpose programming languages to define infrastructure.

It was such a good idea that a few months later, **Terraform** published a preview of its **CDK** to write **Terraform** in **TypeScript** (and I believe they now also support other languages).

With all the pressure for a DevOps culture, this fits really well as it enables developers to use a familiar language to also define infrastructure (it also allow interop between app and infra code)

It supports .NET languages (C#/F#/VB.NET/...), Go, TypeScript, Python.

The ability to use the **Azure management SDKs** is not new, we could always do that in those languages, but **Pulumi** manages all the resources and dependencies for us.

You can specify the resources to create in a declarative way; you just build a list of stuff to create and **Pulumi** works out dependencies, changes and everything else for you.

It also features an encryption capability for secrets in the state; can use external providers to encrypt the content moving the responsibility of securing the state more towards the tool.

The only downside may be that, for system administrators, picking up a programming language may have a steeper learning curve than learning **HCL** or **Bicep** and you are more likely to find ops talents on the job market that know **Terraform** rather than **C#**.

The support story is similar to **Terraform**, **Pulumi** offers a paid plan for storage, but the actual tool is open-source and community-supported.

One feature that **Terraform** has, but it's missing in **Pulumi** is the ability to plan the changes to an output file and then apply that file later; this eliminates the risk of race conditions if you plan and deploy independently and I find it really useful in CD pipelines.

There is no roadmap for it at the moment, but if they will support **PowerShell** as a language in the future, that may remove the need for sysadmins to learn a new language and I believe it could ramp up its adoption.

```csharp
using Pulumi;
using Pulumi.AzureNative.Resources;
using Pulumi.AzureNative.Storage;
using Pulumi.AzureNative.Storage.Inputs;

class MyStack : Stack
{
    public MyStack()
    {
        var resourceGroup = new ResourceGroup("rg-pulumi");

        new StorageAccount("sa", new StorageAccountArgs
        {
            ResourceGroupName = resourceGroup.Name,
            Sku = new SkuArgs
            {
                Name = SkuName.Standard_LRS
            },
            Kind = Kind.StorageV2,
            EnableHttpsTrafficOnly = true
        });
    }
}
```

### Pros

- The **AzureNative** provider is always up-to-date with new **Azure** resources and features
- If you are a developer, you do not need to learn a new language
- It gives you the full power of a real programming language, whatever you can do in C# (or Python etc...) you can do in a **Pulumi** project
- Great CLI, quiet in the output by default

### Cons

- Niche, less likely to find experts and documentation is poor
- More hostile to pick up for sysadmins than Terraform
- No support packages
- No output to the planning phase

## Pulumi.FSharp.Extensions

I wanted to take **Pulumi** a step further; I love the technology, but I still do not like the verbosity of **C#** and, using **Pulumi** in **F#** was ugly, it is not made for this and you end up with code that looks like this to mimic property initialisation in **C#**:

```fsharp
let infra () =
    let resourceGroup = ResourceGroup("rg-pulumi")

    StorageAccount("sa", StorageAccountArgs(
        ResourceGroupName = resourceGroup.Name,
        Sku = SkuArgs(
            Name = SkuName.Standard_LRS
        ),
        Kind = Kind.StorageV2,
        EnableHttpsTrafficOnly = true
    ));
```

So I decided to write an extension to use Pulumi, but make it look even better than **Terraform** in **F#** using computational expressions:

```fsharp
let rg =
    resourceGroup {
        name                   "rg-pulumi"
    }

let sa =
    storageAccount {
        name                   "sa"
        resourceGroup          rg.Name
        accountReplicationType SkuName.Standard_LRS
        accountTier            Kind.StorageV2
        enableHttpsTrafficOnly true
    }
```

### Bird's eye view and lines of code

Purely looking at the core of the template, probably **Pulumi** in **C#** is the shorter template, but, being fair, you also have an external **Pulumi.yaml** file with project and state configuration (which is included in my **Terraform** example) and language-specific files such as project files (csproj), solution etc... **Terraform** and **Bicep** are also quite short and the **ARM template** is the most verbose as anticipated.

## Other options

### PSArm

A new option that recently came up (announced a few days before this blog post) is: **PSArm**

I have not had a chance to play with it and I will update this article later, but **PSArm** seems to answer the prayers of many sysadmins fed up with learning new languages;

It is a way of writing **ARM templates** using idiomatic **PowerShell** which is familiar language to ops.

It sounds quite appealing to those who want to reuse existing skills to embrace **IaC** and **PowerShell** is almost as flexible as a programming language which may help overcome some limitations of **Terraform**.

### Farmer

Honourable mention goes to **Farmer** which lets you write nice-looking **F#** that generates **ARM templates**; I have not played with it at all, but I will try to see if there is any advantage in using it over **Pulumi** (and **Pulumi.FSharp.Extensions** if you want it to look pretty). Bear in mind that, generating **ARM** it means that it works only on **Azure**.

### Azure CLI (Bash/PS) and PowerShell (Az module)

In my opinion a less honourable mention, many infrastructure engineers adopted this method because they did not know any better, I personally see no benefits in using this approach as it moves a massive burden towards the engineer;

You have to worry about dependencies
You have to worry about error handling
You have to make sure it is idempotent (and cmdlets not always are)
...

I would not recommend this option or the ARM templates route, I believe that there is no compelling reason to write **IaC** in this way. Please let me know in the comments if you have a good use case and I will happily update the article.

### Comparison table

![Comparison table](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5k6jhs8aqclbqbs7rh8b.png)

This is a comparison table where I evaluate features for each tool.

## Features comparison

### Declarative

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xto2uqy9iokwjxarkcxl.png)

That is the difference between getting into a shop and asking for a "chocolate cake with cream filling and 30 candles" and telling the baker: OK, now shake the eggs, mix with sugar, add milk etc...

Writing declarative code means telling the system what you want, not how to do it. You lose control in favour of a feature-rich simplicity. That results also in less verbosity.

Most of the options are declarative, you can define resources in whichever order you prefer and the tools will work out dependencies and parallelisation for you. They will also manage retries, error handling and so on without having to explicitly code for it.

The only options that are imperative are **Azure CLI** and **PowerShell** (or directly using the REST API/SDK) to create the resources. I would not recommend this to anyone. It is worth upskilling (if you are a sysadmin) to understand **Pulumi** or **Terraform** and avoid PowerShell (or potentially try **PSArm** or wait for **Pulumi** to support **PowerShell**)

### Idempotency

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xl6icw0nz8iwqq4mmqxx.png)

All the options *can* be idempotent; idempotency means that you can rerun the same deployment as many times as you like and, as long as the resources are unchanged, it will do nothing;

if a resource drifted away from the configuration or does not exist, it will be picked up.

**Azure CLI** and **PowerShell** are in yellow as you can still achieve this, but you have to code against it in certain cases; many **cmdlets** and **AzCLI** commands will be idempotent, but there is no guarantee.

### Fallback mechanism

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gor3vq8t8k9alt4vztc3.png)

Both **Terraform** and **Pulumi** can include **ARM templates** in their code;

if a resource is not supported yet, you can temporarily use an **ARM template** and then update it later when the provider gets updated. **Pulumi** is unlikely to be out of date as it auto-generates from the **Azure REST API**; the folks there just need to kick off another build and in a matter of minutes a new **Pulumi** library is ready with the new **Azure** resources supported.

**ARM/Bicep** are updated immediately, **AzCLI/PS** almost immediately. I have never seen a resource available only in the **REST API** and not in all those.

### Modularisation

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qs5g8u99ijk61wd1sy6o.png)

**ARM** has an awful way of modularising templates, **Bicep** improves that significantly.

**Terraform** works nicely with modules and also supports modules directly from a **Git** repository and **Pulumi** is as good as the language you pick (which is very good for all the languages); you can use **NuGet** packages in **.NET**, **npm** packages with TypeScript, I presume **pip** with **Python** and I am sure that also applies to **Go** with its own package system.

### Legacy deployments

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kzp2imc0jf804x7pqyp5.png)

**ARM** is not that flexible, but it won't matter at all as most of the time it will not need to use any legacy code; I consider **ARM** itself the "legacy".

**Bicep** has an automated tool to convert from **ARM**.

From **Terraform** you can invoke commands locally (including **PowerShell/Bash** scripts)

**Pulumi**, again, can do whatever **TS/C#/Python/Go** can do; which is pretty much everything your computer can do, including invoking **Terraform**, **REST APIs**, buy a **pizza** on every deployment using your favourite pizza place APIs, feed your cat in your smart home or play a fanfare when a resource is created...

### Supportability

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/umg3fz7nj3lexenl371b.png)

**ARM/Bicep** are supported by **Microsoft**, not much else to say there, it is a big plus

**Terraform** and **Pulumi** by the community, but you can get a **paid** support plans if you use their storage.

**AzCLI/PS** you get the obvious support for the tools, but, if your custom code goes wrong, you're on your own and it will be mostly your custom code that will fail.

### Error handling/Plan/Clean up

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pfko4h955wre4h2cnwil.png)

This is all managed for your by all the tools, except **CLI/PS** where you have to look after this yourself; write conditional code and specify retries and what to do if it all goes bad.

###### This is a live article, I will try and keep it up to date with the new development and to complete the missing bits, if you want to suggest a change, please submit a pull request to [this](https://github.com/UnoSD/Blog) repository.

###### Opinions expressed are solely my own and do not express the views or opinions of my employer

###### Cover image: "A visual representation of the DevOps workflow" by [Kharnagy](https://commons.wikimedia.org/wiki/User:Kharnagy) (edited) is licensed under CC BY-SA 4.0, .
