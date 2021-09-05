---
title: 'Terraform: Multi-cloud demo – Part 2 – Terraform Remote State on Azure'
date: Sat, 03 Apr 2021 08:57:31 +0000
draft: true
tags: ['Azure', 'azure', 'Blog', 'demo', 'GitHub', 'Remote State', 'remote-state', 'Terraform', 'Terraform', 'training']
author: "Dave"
toc: true
featured_image: "image/title/terraform-training-demo2-azure-remote-state.png"
---

So we have setup our source control (GitHub) for team collaboration [(Part 1)](https://davehart.co.uk/index.php/2021/03/26/terraform-multi-cloud-demo-part-1-foundations/) , next we should consider the Terraform state file.

By Default Terraform will create a local state file (terraform.tfstate), this does not work when collaborating as each person needs to have the latest version of the state data prior to performing any Terraform actions. In addition you need to ensure that nobody else is running Terraform at the same time. Having a remote state helps mitigate these issues.

There are a number of different locations that Terraform supports for storing remote state, we will look at using Azure Blob Storage in this demonstration (I used terraform cloud in [this article](https://davehart.co.uk/index.php/2020/09/19/terraform-aws-wordpress/) if you are interested). This natively allows state locking and consistency checking. In addition we will use a SAS token to provide authentication to collaborators.

Root module
-----------

We are going to setup this environment with re-usable repeatable code in mind. The working directory when we start defining our resources, variables and outputs is referred to as the root module. Through this demo we will be using modules from the terraform registry and we will make some of our own. Lets start with the root module;

![](https://davehart.co.uk/wp-content/uploads/2020/09/information-1015297_1280-1024x1024.jpg)

Modules are a wrapper around a collection of Terraform resources. Using modules allows you to break your code into smaller blocks which can make reading code easier and aid in troubleshooting. In addition a module allows for easier reuse of code blocks.

All modules tend to have the same construct, a main.tf , output.tf and the variables.tf. Terraform will process any file with a .tf extension and it does all the heavy lifting working out the run order, dependencies etc. You could call the files whatever you want but lets standardise.

### main.tf

First off we need to define our provider, we will be using the [azurerm](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs) provider;

```
provider "azurerm" {
  features {}
}
```

Pretty simple. The features part need not have any values but the provider does not work without it declared. I will need to run 'a_z login_' in my console to authenticate with my Azure subscription once I start creating resources but for now that is all we need.

At this point we could start defining all of our Azure resource blocks but lets drop it all into a module;

### module : azure\_remote\_state

As mentioned above, code re-use is key so we could take a look at the [terraform registry](https://registry.terraform.io/) and see if there is already a module we could use for our azure remote state. Well there is but I am not using it for two reasons. Firstly it does not give me everything I want and secondly it is a good excuse to create our own module and learn something. We will be consuming existing modules from the registry later on in the demo.

Any module we create needs to be located in a modules/module\_name folder in our root module location. The module folder can be called anything you like. For our example we will use "remote\_state". Under this folder we create the standard constructs (main.tf, variables.tf and output.tf)

![](https://davehart.co.uk/wp-content/uploads/2021/03/image-9.png)

We will start off in the module main.tf and construct the Azure components we need for our remote state. But first, lets define a terraform configuration in our module. At a minimum we should be specifying the required version based on what we used to develop this example.

```
terraform {
    required\_version = ">=0.14.8"
}
```

Now we can move onto our remote state configuration. The remote state will need;

Azure resource group > Azure storage account > container in the storage account

Lets start with the resource group. The section headings will reflect the names defined in the azurerm provider.

#### azurerm\_resource\_group

We need to define a name for the resource group a location and perhaps some tags at a minimum. The values for these should be variables and can be changed on the module calling block in the root main.tf

```
\# Define the resource group for hosting the storage account
resource "azurerm\_resource\_group" "demo" {
  name     = var.rg\_name                   # We need to declare this as a variable in the modules variables.tf
  location = var.rg\_location               # We need to declare this as a variable in the modules variables.tf
  tags = merge(
    var.tags, {                            # We need to declare this as a variable in the modules variables.tf
      Name = "var.rg\_name"
  })
}
```

![](https://davehart.co.uk/wp-content/uploads/2020/09/information-1015297_1280-1024x1024.jpg)

merge Function

We have used the merge function in the tags block. This function will return a single map or object from all arguments. More details [here](https://www.terraform.io/docs/language/functions/merge.html).

Seeing as we already require variables defined, we will add them to the modules variables.tf as we go.

```
variable "rg\_name" {
  type    = string
  default = "demo"
}

variable "rg\_location" {
  description = "azure location"
  type        = string
  default     = "UK West"
}

variable "tags" {
  description = "resource tags"
  type        = map(any)
  default = {}
}
```

Now we have a resource group to host our resource we can define the storage account.

#### azurerm\_storage\_account

One of the criteria for a Azure storage account is that the name is unique within Azure. No two storage accounts can have the same name. Fortunately Terraform has a random provider, we will use the random\_integer resource from this provider to help us generate a random name for our storage account.

```
resource "random\_integer" "stacc\_num" {
  min = var.random\_min                        # We need to declare this as a variable in the modules variables.tf
  max = var.random\_max                        # We need to declare this as a variable in the modules variables.tf
}

resource "azurerm\_storage\_account" "stacc" {
  name                     = "${lower(var.stacc\_name\_prefix)}${random\_integer.stacc\_num.result}"     # We need to declare this as a variable in the modules variables.tf
  resource\_group\_name      = azurerm\_resource\_group.demo.name
  location                 = var.rg\_location     # We have already declared this variable
  account\_tier             = var.acc\_tier        # We need to declare this as a variable in the modules variables.tf
  account\_replication\_type = var.acc\_rep\_type    # We need to declare this as a variable in the modules variables.tf
}
```

Lets add the variables;

```
variable "random\_min" {
    description = "minimum value for random number range"
    type = number
    default = 10000
}
variable "random\_max" {
    description = "maximum value for random number range"
    type = number
    default = 99999
}
variable "stacc\_name\_prefix" {
    description = "prefix for the azure storage account name - random value will be added as suffix"
    type = string
    default = "example"
}
variable "acc\_tier" {
  description = "azure storage tier"
  type        = string
  default     = "standard"
}
variable "acc\_rep\_type" {
  description = "azure replication type"
  type        = string
  default     = "LRS"
}
```

#### azurerm\_storage\_container

The container is used to organise your blobs. Think of it as a directory structure in a file system. There are no limits to the number of containers or the number of blobs a container can store.

```
resource "azurerm\_storage\_container" "ct" {
  name                 = var.container\_name      # We need to declare this as a variable in the modules variables.tf
  storage\_account\_name = azurerm\_storage\_account.stacc.name
}
```

And the variables

```
variable "container\_name" {
  description = "azure storage account container name"
  type        = string
  default     = "demo"
}
```

#### azurerm\_storage\_account\_sas

When sharing the remote state we want some degree of access management and not have something publicly accessible. Once we have created the storage account we can obtain a Shared Access Signature (SAS Token) which allows fine-grained, ephemeral access control to various aspects of the storage account. This is defined as a data source type and not a resource type as we have been using previously. There are a lot of setting here so I will link to the [official docs](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/storage_account_sas) which you can read at your own leisure.

```
data "azurerm\_storage\_account\_sas" "state" {
  connection\_string = azurerm\_storage\_account.stacc.primary\_connection\_string
  https\_only        = true
  resource\_types {
    service   = true
    container = true
    object    = true
  }
  services {
    blob  = true
    queue = false
    table = false
    file  = false
  }
  start = var.sas\_start     # We need to declare this as a variable in the modules variables.tf
  expiry = timeadd(var.sas\_start, var.sas\_timeadd) # We need to declare this as a variable in the modules variables.tf

  permissions {
    read    = true
    write   = true
    delete  = true
    list    = true
    add     = true
    create  = true
    update  = false
    process = false
  }
}
```

All of the settings could be created as variables but for our use case it's not required. Lets add the two we do need which will define the length of time the sas token is valid (before it expired). We can control these from the root module when calling this module.

```
variable "sas\_start" {
  description = "Start time for SAS token expiration"
  type        = string
  default     = "timestamp()"
}

variable "sas\_timeadd" {
  description = "time to add to sas token start time which calculates the expiration duration"
  type        = string
  default     = "24h"
}
```

#### null\_resource / local-exec provisioner

When we change from local to remote state we will add a backend configuration to our root module and perform a terraform init feeding in the variables required for the new backend. What we will do now is create an output file containing this information in the required format. We can feed this in as the -backend-config parameter when we run the terraform init

The required variables are;

*   storage\_account\_name
*   container\_name
*   key (we will use the Terraform default terraform.tfstate)
*   sas\_token

It goes without saying this information needs to be securely stored and securely shared.

To generate this output we are going to utilise the Terraform null\_resource and local-exec provisioner.

![](https://davehart.co.uk/wp-content/uploads/2020/09/information-1015297_1280-300x300.jpg)

The null\_resource allows you to run a provisioner that is not directly associated with a specific resource. null\_resources are treated as normal resources but they don't do anything

The local-exec provisioner invokes a local executable after a resource is created.

```
resource "null\_resource" "post-deploy" {
  depends\_on = \[azurerm\_storage\_container.ct\]     # We need to ensure the container exists before providing the output
  provisioner "local-exec" {
  command = <<EOT
  echo 'storage\_account\_name = "${azurerm\_storage\_account.stacc.name}"' >> ${var.sas\_output\_file} # variable we need to define
  echo 'container\_name = "tf-state"' >> ${var.sas\_output\_file}
  echo 'key = "terraform.tfstate"' >> ${var.sas\_output\_file}
  echo 'sas\_token = "${data.azurerm\_storage\_account\_sas.state.sas}"' >> ${var.sas\_output\_file}
  EOT
}
}
```

Just the one variable for this one which is the output file

```
variable "sas\_output\_file" {
  description = "output file containing sas details for remote state"
  type        = string
  default     = "example.txt"
}
```

That finishes off the module configuration. The only output we need is being handled by our null\_resource so we don't really need the output.tf but I may use it so will leave it there for now.

Next we look at calling the module and providing all those values we defined as variables.

Back to the root module main.tf

root - module
-------------

so far our root module is looking a bit light. We have only added the Azure provider. Lets now add a module block so we can utilise the remote state module we just created.

```
module "azure\_remote\_state" {
  source            = "./modules/remote\_state"               # Where it the module located
  random\_min        = 100000                                 # Minimum value for our random integer
  random\_max        = 999999                                 # Maximum value for our random integer
  rg\_name           = "demo-rg"                              # Name for the Azure Resource Group
  rg\_location       = "UK West"                              # Geographic location for our Azure Resource Group
  stacc\_name\_prefix = "demonstration"                        # Prefix for our storage account - Random integer will be appended
  acc\_tier          = "standard"                             # Azure Storage Account Tier
  acc\_rep\_type      = "LRS"                                  # Azure Storage Redundancy
  container\_name    = "tf-state"                             # Azure Storage Account Container Name
  sas\_start         = timestamp()   #"2021-03-30T07:00:00Z"  # Starting time for the SAS Token including req. format
  sas\_timeadd       = "48h"                                  # Duration from starting time when SAS Token will expire
  sas\_output\_file   = "sas-remote-state.txt"                 # Output file containing all the information we need to move to remove state
}
```

That should be enough for us to provision our remote state infrastructure. All goes well we will have an input file we can use when we change the state to remote. Lets see if it works!

### Terraform init

We will first initialise our Terraform configuration so Terraform can prepare our working directory for its use.

![](https://davehart.co.uk/wp-content/uploads/2021/03/terraform_init_azure_remote_state.gif)

Terraform output for the init phase

No issues with the 'terraform init' so lets try 'terraform plan' next including a plan file.

![](https://davehart.co.uk/wp-content/uploads/2021/03/image-10-1024x193.png)

This is just to show you what happens if you forget to login to azure first, its pretty descriptive :)

Now that I have logged into Azure.....

The plan..

![](https://davehart.co.uk/wp-content/uploads/2021/03/terraform_plan_azure_remote_state_azlogin_required.gif)

Terraform output for the plan phase

Plan is looking good.

![](https://davehart.co.uk/wp-content/uploads/2021/03/image-13.png)

terraform plan -out demo.tfplan

Lets proceed with the 'terraform apply' using the plan file and then take a look at our Azure Resource groups

![](https://davehart.co.uk/wp-content/uploads/2021/03/terraform_apply_azure_remote_state.gif)

Terraform output for the apply phase

![](https://davehart.co.uk/wp-content/uploads/2021/03/image-12.png)

terraform apply demo.tfplan

Lets see if the backend configuration file with the SAS Token details was generated;

![](https://davehart.co.uk/wp-content/uploads/2021/03/image-11.png)

sas-remote-state.txt generated using null\_resource and local-exec

Everything is looking good. Our Azure infrastructure is deployed ready to hold our terraform state. We can now enable remote state!

Enable remote state
-------------------

There are two parts to this process, first we need to update the root module to use a terraform configuration block (same as we did for the version dependency in our module) and then run terraform init with some backend parameters. Lets first update the root module main.tf

All we need to define in the root module is the following;

```
terraform {
    backend "azurerm" {
    }
}
```

This is telling terraform to use the azurerm provider to connect to the backend, All it needs now are the backend configuration information which we provide from our sas-remote-state.txt file when we initialise the remote state.

![](https://davehart.co.uk/wp-content/uploads/2021/03/image-14-1024x270.png)

terraform init using the previously generates backend configuration file

You can see that terraform has located the remote backend and seen there is no state there presently. We will answer yes to copy the local state to the remote backend.

![](https://davehart.co.uk/wp-content/uploads/2021/03/image-15.png)

terraform init using the previously generates backend configuration file

it we check the Azure storage account container we should see our state file;

![](https://davehart.co.uk/wp-content/uploads/2021/03/image-16-1024x389.png)

Azure portal showing the terraform.tfstate (remote state)

![](https://davehart.co.uk/wp-content/uploads/2021/03/terraform_init_azure_remote_state-1.gif)

Moving to remote state

Source control
--------------

Throughout the process we should be committing our changes to our VCS and making any changes to the README.md or .gitignore file.

Following on from this exercise I have added an exclusion to my .gitignore file to ensure the storage account configuration is not in the public space.

```
\# Text file which may contain storage account credentials
\*.txt
```

It is a good idea to then clear the git cache when making changes to tracked or untracked files in the .gitignore file. The following command sequence was used to clear the cache and update all our changes.

```
git rm -r --cached .   
git add .              
git commit -m "commit message"
git push
```

Summary
-------

That ends this part of the demonstration. We have successfully defined our Azure configuration for hosting our Terraform state file and copied our state over by initialising the terraform configuration. If you want to move back to a local state for testing you can change the 'azurerm' part of the backend configuration to 'local'. Once you run terraform init again (without the backend configuration file) you will be asked to copy the file locally.

In [part 3](https://davehart.co.uk/index.php/2021/04/09/terraform-multi-cloud-demo-part-3-aws-infrastructure/) we will start deploying the AWS infrastructure components.

**Disclaimer: You may incur costs if you follow along with the exercise. Use at your own risk!**