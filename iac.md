Infrastructure as code, **what** is it and **why** should we invest and introduce further complexity in our process?

**IaC** is the process of defining IT infrastructure in a language that can be interpreted by tools in order to create and configure resources (both in cloud and on-premises environments).

There are numerous advantages to using IaC, here are few examples:

- **Automation** Scripted deployments reduce surface for human error and are faster to execute
- **Reproducibility** Being able to recreate the same environment every time or multiple identical environment based on the same templates
- **Security** Permission to production environments can be granted only to deployment service accounts, reducing risks
- **Costs** Being able to destroy and recreate environments quickly can enable fast de-provisioning of expensive resources when they are not used

# What are the options?

This article is Azure-centric, but most of what will be discussed will also apply to AWS, GCP and other clouds and targets. Cloud-specific systems such as ARM templates/Bicep may be comparable with AWS CloudFormation, GCP Cloud Deployment Manager and to K8s YAML for on-premises targets.

## ARM templates

Azure-native way of scripting resources; the language used is JSON.

I have worked extensively with ARM templates, but I still get an headache when I open one for the first time; JSON is readable, but not intuitive. I am a spoilt child of F# programming, when I see so much boilerplate I cannot focus.

Personal preference aside, I believe it is not concise and requires substantial nesting and a plethora of parenthesis and quotes.

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

### Cons

- Unfriendly language
- Hard to manage modules
- Complexity increases exponentially for large environments

## Bicep

That's pretty much the same as an ARM template, in fact it transpiles to ARM JSON to use the same underlying deployment system, but Bicep addresses the two main downsides of ARM.

One massive advantage over ARM is the language; it's a bespoke DSL, easy to write and understand.

Modularisation is also easier than ARM as it allows to reference other templates in the file system.

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

## Terraform

Terraform is the exceptional attempt at a multi-cloud, human-readable, infrastructure as code tool; a successful attempt, it is quickly becoming the industry standard for cloud deployments.

The HashiCorp tool uses an external state storage, this registers all the resources deployed and enables destruction of the resources removed from the templates and clean up of an entire environment.

Uses a custom DSL called HCL, quite friendly and understandable by anyone at a glance. One major downside is that is a limited language; it covers the basic conditionals, loops, variables (poorly in my opinion).

It's a beautiful solution for simple deployments, but it gets pretty frustrating when attempting to be more clever with the logic.

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

- Industry standard as of 2021
- Language and CLI are incredibly easy to understand and to use

### Cons

- HCL has its limits and simple logic can turn into complex templates
- Storage of the state is in plain text, including secrets; responsibility for securing it lies with the user
- Tooling and code completion, compile-time checks

## Pulumi

[Dulcis in fundo...](https://en.wiktionary.org/wiki/dulcis_in_fundo) my favourite IaC tool as of today. The people at Pulumi had a great intuition of using general purpose programming languages to define infrastructure. It was such a good idea that a few months later, Terraform published a preview of its CDK to write Terraform in TypeScript.

With all the pressure for a DevOps culture, this fits really well as it enables developers to use a familiar language to also define infrastructure.

Supports .NET languages (C#/F#/VB.NET/...), Go, TypeScript, Python

The ability to use the Azure management SDKs is not new, we could always do that, but Pulumi manages all the resources and dependencies for us.

You can specify the resources to create in a declarative way; you just build a list of stuff to create and Pulumi works out dependencies, changes and everything else for you.

It also features an encryption capability for secrets in the state; can use external providers to encrypt the content moving the responsibility of securing the state more towards the tool.

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

- 

### Cons

- Niche, less likely to find experts and documentation is poor
- More hostile to pick up for sysadmins than Terraform
- No support packages

## Pulumi.FSharp.Extensions

I wanted to take Pulumi a step further; I love the technology, but I still do not like the verbosity of C# and, using Pulumi in F# was ugly, it is not made for this and you end up with code that looks like this to mimic property initialization in C#:

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

So I decided to write an extension to use Pulumi, but make it look even better than Terraform in F#:

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



## Other options

PSArm Familiar language to system administrators
Farmer
Deployments using Azure CLI scrips and PowerShell

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5k6jhs8aqclbqbs7rh8b.png)

This is a comparison table where I evaluate features for each tool.

### Declarative

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xto2uqy9iokwjxarkcxl.png)

That is the difference between getting into a shop and asking for a "chocolate cake with cream filling and X candles" and telling the rep: shake the eggs, mix with sugar, etc...

All the options are declarative, you can define resources in whichever order you prefer and the tools will work out dependencies and parallelisation for you. They will also handle retries, error handling and so on. You just tell the tool what you want, not how you want it to be created.

The only options that are imperative are using Azure CLI, PowerShell (or directly the REST API/SDK) to create the resources. I would not recommend this to anyone. It is worth upskilling your sysadmins to understand Pulumi or Terraform and avoid PowerShell (or potentially try PSArm or wait for Pulumi to support PowerShell)

### Idempotency

All the options *can* be idempotent; idempotency means that you can rerun the same deployment as much as you like and, as long as the resources are unchaged, it will do nothing; if a resource drifted away from the configuration or does not exist, it will be picked up.

Azure CLI and PowerShell are in yellow as you can still achieve this, but you have to code against it in certain cases; many cmdlets and AzCLI commands will be idempotent, but there is no guarantee.

### Fallback mechanism

Both Terraform and Pulumi can include ARM templates in their code; if a resource is not supported yet, you can temporarily use an ARM template and then update it later when the provider updates. Pulumi is unlikely to be out of date as it auto-generates from the Azure REST API; the folks there just need to kick off another build and in a matter of minutes a new Pulumi library is ready with the new Azure resources supported.

AzCLI/PS/ARM/Bicep are updated almost immediately. I have never seen a resource available only in the REST API and not in all those.

### Modularisation

ARM has an awful way of modularising templates, Bicep XXX, Terraform works nicely with modules and also supports modules directly from a Git repository and Pulumi is as good as the language you pick (which is very good for all the languages), you can use NuGet packages in .NET, npm packages with TypeScript, pip? with Python and whatever Go uses...

### Legacy deployments

ARM/Bicep are not that flexible, but it won't matter most of the time I consider ARM itself the "legacy". From Terraform you can invoke commands locally and Pulumi, again, can do whatever C#/Python/Go can do; which is pretty much everything you computer can do, including invoking Terraform, REST API, buy a pizza on every deployment using your favourite pizza place APIs, feed your cat or play a fanfare...

### Supportability

ARM/Bicep are supported by Microsoft, not much else to say; Terraform and Pulumi by the community, but you can get a paid support plan for Terraform, for Pulumi you cannot.

CLI/PS you get the obvious support for the tools, but, if your custom code goes wrong, you're on your own and it will be mostly your custom code that will fail.

### Error handling/Plan/Clean up

This is all managed for your by all the tools, except CLI/PS where you have to look after this yourself; write conditional code and specify retries and what to do if it all goes bad.

###### Cover image: "A visual representation of the DevOps workflow" by [Kharnagy](https://commons.wikimedia.org/wiki/User:Kharnagy) (edited) is licensed under CC BY-SA 4.0, .
