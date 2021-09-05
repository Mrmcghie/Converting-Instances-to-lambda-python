---
title: 'Terraform: Multi-cloud demo – Part 3 – AWS Infrastructure'
date: Fri, 09 Apr 2021 11:23:40 +0000
draft: true
tags: ['AWS', 'AWS', 'Blog', 'demo', 'modules', 'remote-state', 'Terraform', 'Terraform']
author: "Dave"
toc: true
featured_image: "image/title/terraform-training-demo3-aws-infra.png"
---

It's going well so far. We have our [source control defined and updated](https://davehart.co.uk/index.php/2021/03/26/terraform-multi-cloud-demo-part-1-foundations/) and our Terraform remote state hosted on Azure Storage which we provisioned using Terraform. Let's now move onto provisioning our AWS infrastructure.

Root module
-----------

### main.tf

First off we need to add the [aws](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs) provider;

```
provider "aws" {
  region  = var.region
  profile = var.profile
}
```

You will notice that we have also used a couple of variables with this provider. The first will define our aws region and the second will tell the aws provider which profile to use. As I have numerous AWS account I am working in, I tend to use [named profiles](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html) to define my security tokens. You can add secure tokens in the provider but that is a big no no. Especially as we are using a publicly accessible source control.

If you wanted to work across multiple accounts or regions you could add another aws provider and include an alias key. (e.g. alias = "west1") which has its own configuration. You refer to this then using the 'provider = aws.alias' value in your resources .

Let's add the variables into our root module variables.tf

### variables.tf

```
variable "region" {
  description = "aws region"
  type        = string
  default     = "eu-west-2"
}
variable "profile" {
  description = "aws user profile to utilise"
  type        = string
  default     = "capgemini"
}
```

Terraform has a number of ways of assigning variables values. We have used the default method in the above code block. If we omitted the default, terraform would prompt you when you apply your configuration for the value.

Alternatively we could create a terraform.tfvars file with the value declared, for example

```
region = "eu-west-2"
profile = "capgemini"
```

You will find that Terraform has many ways to handle variables. So [here](https://learn.hashicorp.com/tutorials/terraform/variables?in=terraform/configuration-language) and [here](https://www.terraform.io/docs/language/values/variables.html#variable-definition-precedence) are a few links worth reading; We will use different methods throughout this demo.

module - vpc
------------

For our AWS infrastructure we are going to consume a module from the [terraform registry](https://registry.terraform.io/). If we navigate to the module and filter by AWS, the one we want is on the top of the list

![](https://davehart.co.uk/wp-content/uploads/2021/04/image-1024x375.png)

This is a [module](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest) verified by Terraform and provided by AWS. The module provides the below provisioning instructions.

```
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "2.77.0"
  # insert the 49 required variables here
}
```

The module has over 400 inputs available so I am just going to show what we need for our deployment. I decided to use a data source for the availability zones. This allows us to fetch data from our aws provisioner relating to our region.

```
module "vpc" {
  source                   = "terraform-aws-modules/vpc/aws"
  version                  = "2.77.0"
  cidr                     = "10.0.0.0/16"
  azs                      = data.aws\_availability\_zones.available.names
  private\_subnets          = \["10.0.2.0/28", "10.0.4.0/28"\]
  public\_subnets           = \["10.0.1.0/28"\]
  enable\_dns\_hostnames     = true
  enable\_nat\_gateway       = true
  single\_nat\_gateway       = true
  public\_subnet\_tags = {
    Name = "${var.env\_prefix}-public"
  }
  tags = var.tags
  vpc\_tags = {
    Name = "${var.env\_prefix}-VPC"
  }
  private\_subnet\_tags = {
    Name = "${var.env\_prefix}-private"
  }
  igw\_tags = {
    Name = "${var.env\_prefix}-igw"
  }
   nat\_gateway\_tags = {
    Name = "${var.env\_prefix}-natgw"
  }
}

data "aws\_availability\_zones" "available" {
  state = "available"         # Return all available az in the region
}
```

This is a pretty bare bones configuration. Lets break it down ignoring the provisioning defaults (source and version)

```
cidr                     = "10.0.0.0/16"                               #VPC CIDR Range
azs                      = data.aws\_availability\_zones.available.names # data block providing available az in region
private\_subnets          = \["10.0.2.0/28", "10.0.4.0/28"\]              # List of private subnets
public\_subnets           = \["10.0.1.0/28"\]                             # List of public subnets
enable\_dns\_hostnames     = true                                        # Allow our instances to have name resolution
enable\_nat\_gateway       = true                                        # Create a nat gateway
single\_nat\_gateway       = true                                        # For the demo I am only having 1 nat gateway
```

I am also defining a number of tags for each of the items using a combination of a tag variable and/or an env\_prefix variable. We will define them in the root module variables.tf along with the region and profile variables.

```
variable "tags" {
  description = "resource tags"
  type        = map(any)
  default = {
    CostCentre : "common"
    Project : "infra"
    Description : "demo"
    Owner : "dave.hart"
  }
}
variable "env\_prefix" {
  description = "prefix for naming"
  type        = string
  default     = "demo"
}
```

![](https://davehart.co.uk/wp-content/uploads/2020/09/information-1015297_1280-1024x1024.jpg)

type = map

The map variable type represents a collection of keys and values (key value pairs)

That should cover our network infrastructure. Next up we start considering our two instances and what requirements we have for them.

Our two AWS EC2 instances will need a few different configuration components. I want them to be accessible via AWS systems manager so we can establish a session if required. We will achieve this by creating an instance profile/role which can be attached at the time of provisioning. Next the instances will need some security groups so we can access them and last of all we want to deploy the instances. When we deploy the EC2 instances we will be applying some user\_data code to deploy our software and configure them. This is not necessarily a great use case for terraform as a single change to the user\_data would result in the instance being redeployed. For software deployment and desired state there are tools more suited such as ansible or puppet to name a few. Seeing as this is a Terraform demo we will stick with using Terraform but you should consider different tools for any configuration management.

module : security-groups
------------------------

We will use an existing module from the terraform registry for our security groups. The second module in the registry under the AWS provisioner is for security groups;

![](https://davehart.co.uk/wp-content/uploads/2021/04/image-1.png)

security-group module

If we take a look at the provisioning instructions it implied we do not need a lot of configuration;

```
module "security-group" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "3.18.0"
  # insert the 2 required variables here
}
```

We need a few more than the 2 required variables. We will be defining two SG, one for the proxy server and one for the Prometheus server. Here is our custom rules for the security groups with comments inline.

```
module "security-group-prometheus" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "3.18.0"
  name = "SGAllowPrometheus"                         # Name of the Security group
  vpc\_id = module.vpc.vpc\_id                         # VPC Id using the value from the vpc module
  tags = merge(                                      # Tagging - add the tag variables and merge with name
    var.tags, {
      Name = "SgAllowPrometheus"
  })
  ingress\_with\_cidr\_blocks = \[                       # SG custom ingress rules for Prometheus server which is in 
    {                                                # a Private subnet
      from\_port   = 9090
      to\_port     = 9100
      protocol    = "tcp"
      cidr\_blocks = "0.0.0.0/0"
    },
    {
      from\_port   = 9182
      to\_port     = 9182
      protocol    = "tcp"
      cidr\_blocks = "0.0.0.0/0"
    },\]
    egress\_with\_cidr\_blocks = \[                      # SG Egress rules for Prometheus server
    {
      from\_port   = 0
      to\_port     = 0
      protocol    = "-1"
      cidr\_blocks = "0.0.0.0/0"
    },
  \]
}

module "security-group-proxy" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "3.18.0"
  name = "SgAllowProxy"                             # Name of the Security group
  vpc\_id = module.vpc.vpc\_id                        # VPC Id using the value from the vpc module
  tags = merge(                                     # Tagging - add the tag variables and merge with name
    var.tags, {
      Name = "SgAllowProxy"
  })
  ingress\_cidr\_blocks = \["0.0.0.0/0"\]               # SG predefined ingress rules for Proxy server which is in 
  ingress\_rules = \["http-80-tcp"\]                   # a public subnet
  egress\_with\_cidr\_blocks = \[
    {
      from\_port   = 0
      to\_port     = 0
      protocol    = "-1"
      cidr\_blocks = "0.0.0.0/0"
    },
  \]
}
```

Now we need to define our EC2 instance profile.

Module : instance\_profile
--------------------------

This will be a module we create, so lets start off creating a folder under the modules folder for our configuration. We will call this one modules/instance\_profile

next we can define our modules main.tf and add the terraform configuration block defining the minimum required version and the resources we need for the instance profile/role.

```
terraform {
    required\_version = ">=0.14.8"
}
                                   
resource "aws\_iam\_role" "demo-ssm-role" {               # Define our resource role for systems manager
  name = var.role\_name                                  # the role name from a variables
  assume\_role\_policy = jsonencode({                     # use jsonencode to format the role policy to a string
    Version = "2012-10-17"
    Statement = \[
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Sid    = ""
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    \]
  })
  tags = merge(
    var.tags, {
      Name = var.tag\_name
  })
}

resource "aws\_iam\_instance\_profile" "ec2\_profile" {
  name = var.ip\_name                                      # variable to define the instance profile name
  role = aws\_iam\_role.demo-ssm-role.name                  # which role is associated with the instance profile
}

resource "aws\_iam\_role\_policy\_attachment" "ssm-attach" {  # define which existing policies to attach to the the role
  role       = aws\_iam\_role.demo-ssm-role.name            # name of the role
  count      = length(var.policy\_arns)                    # loop through each policy arn defined in the variable
  policy\_arn = var.policy\_arns\[count.index\]
}
```

Now lets define our variables.tf for the instance\_profile module

```
variable "ip\_name" {
    type = string                # instance profile name
    default = ""
}
variable "role\_name" {           # role name
    type = string
    default = ""
}
variable "tag\_name" {            # tag name key value
    type = string
    default = ""
}
variable "policy\_arns" {         # list of arns to attach to the role
  description = "arns to add to ec2 role"
  type        = list(any)
  default     = \[""\]
}
variable "tags" {                # tags to apply
  description = "resource tags"
  type        = map(any)
  default = {}
}
```

For this module we need to know the instance profile name so we can use it for our EC2 instances role attachment. To retrieve this value we will define an output value in the output.tf file within the module path;

```
output "iam\_instance\_profile\_name" {
    value = aws\_iam\_instance\_profile.ec2\_profile.name
}
```

Next we need to call this module from our root module defining the few required values we setup;

```
module "instance\_profile" {
  source      = "./modules/instance\_profile"                             # where we created our configuration
  ip\_name     = "demo\_ec2\_profile"                                       # instance profile name
  role\_name   = "demo\_ssm\_role"                                          # role name
  policy\_arns = \["arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"\] # list of policy arn to attach
  tag\_name    = "demo-ec2-ssm-policy"                                    # the tag key name value
  tags        = var.tags                                                 # pass the root module tag variable values
}
```

Security groups and instance profile defined.

Summary
-------

It's getting a bit long so lets wrap up for now. We have successfully defined our AWS Network components, security groups and instance profile ready for our instances.

In [part 4](https://davehart.co.uk/index.php/2021/04/09/terraform-multi-cloud-demo-part-4-aws-instances/) we will continue to deploy our AWS EC2 instances

Disclaimer: You may incur costs if you follow along with the exercise. Use at your own risk!