---
title: 'Terraform: Multi-cloud demo – Part 3 – AWS Infrastructure'
date: Fri, 09 Apr 2021 11:23:40 +0000
draft: false
tags: ['AWS', 'AWS', 'Blog', 'demo', 'modules', 'remote-state', 'Terraform', 'Terraform']
author: "Dave"
toc: true
featured_image: "image/title/terraform-training-demo3-aws-infra.png"
---

It's going well so far. We have our [source control defined and updated](/post/terraform-multi-cloud-demo-part-1-foundations/) and our Terraform remote state hosted on Azure Storage which we provisioned using Terraform. Let's now move onto provisioning our AWS infrastructure.

Root module
-----------

### main.tf

First off we need to add the [aws](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs) provider;

```HCL
provider "aws" {
  region  = var.region
  profile = var.profile
}
```

You will notice that we have also used a couple of variables with this provider. The first will define our aws region and the second will tell the aws provider which profile to use. As I have numerous AWS account I am working in, I tend to use [named profiles](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html) to define my security tokens. You can add secure tokens in the provider but that is a big no no. Especially as we are using a publicly accessible source control.

If you wanted to work across multiple accounts or regions you could add another aws provider and include an alias key. (e.g. alias = "west1") which has its own configuration. You refer to this then using the 'provider = aws.alias' value in your resources .

Let's add the variables into our root module variables.tf

### variables.tf

```HCL
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

```HCL
region = "eu-west-2"
profile = "capgemini"
```

You will find that Terraform has many ways to handle variables. So [here](https://learn.hashicorp.com/tutorials/terraform/variables?in=terraform/configuration-language) and [here](https://www.terraform.io/docs/language/values/variables.html#variable-definition-precedence) are a few links worth reading; We will use different methods throughout this demo.

module - vpc
------------

For our AWS infrastructure we are going to consume a module from the [terraform registry](https://registry.terraform.io/). If we navigate to the module and filter by AWS, the one we want is on the top of the list

{{< figure src="image" >}}

This is a [module](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest) verified by Terraform and provided by AWS. The module provides the below provisioning instructions.

```HCL
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "2.77.0"
  # insert the 49 required variables here
}
```

The module has over 400 inputs available so I am just going to show what we need for our deployment. I decided to use a data source for the availability zones. This allows us to fetch data from our aws provisioner relating to our region.

```HCL
module "vpc" {
  source                   = "terraform-aws-modules/vpc/aws"
  version                  = "2.77.0"
  cidr                     = "10.0.0.0/16"
  azs                      = data.aws_availability_zones.available.names
  private_subnets          = ["10.0.2.0/28", "10.0.4.0/28"]
  public_subnets           = ["10.0.1.0/28"]
  enable_dns_hostnames     = true
  enable_nat_gateway       = true
  single_nat_gateway       = true
  public_subnet_tags = {
    Name = "${var.env_prefix}-public"
  }
  tags = var.tags
  vpc_tags = {
    Name = "${var.env_prefix}-VPC"
  }
  private_subnet_tags = {
    Name = "${var.env_prefix}-private"
  }
  igw_tags = {
    Name = "${var.env_prefix}-igw"
  }
   nat_gateway_tags = {
    Name = "${var.env_prefix}-natgw"
  }
}

data "aws_availability_zones" "available" {
  state = "available"         # Return all available az in the region
}
```

This is a pretty bare bones configuration. Lets break it down ignoring the provisioning defaults (source and version)

```HCL
cidr                     = "10.0.0.0/16"                               #VPC CIDR Range
azs                      = data.aws_availability_zones.available.names # data block providing available az in region
private_subnets          = ["10.0.2.0/28", "10.0.4.0/28"]              # List of private subnets
public_subnets           = ["10.0.1.0/28"]                             # List of public subnets
enable_dns_hostnames     = true                                        # Allow our instances to have name resolution
enable_nat_gateway       = true                                        # Create a nat gateway
single_nat_gateway       = true                                        # For the demo I am only having 1 nat gateway
```

I am also defining a number of tags for each of the items using a combination of a tag variable and/or an env_prefix variable. We will define them in the root module variables.tf along with the region and profile variables.

```HCL
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
variable "env_prefix" {
  description = "prefix for naming"
  type        = string
  default     = "demo"
}
```

{{< imgproc info Resize "100x" >}}

_**type = map : The map variable type represents a collection of keys and values (key value pairs)**_

That should cover our network infrastructure. Next up we start considering our two instances and what requirements we have for them.

Our two AWS EC2 instances will need a few different configuration components. I want them to be accessible via AWS systems manager so we can establish a session if required. We will achieve this by creating an instance profile/role which can be attached at the time of provisioning. Next the instances will need some security groups so we can access them and last of all we want to deploy the instances. When we deploy the EC2 instances we will be applying some user_data code to deploy our software and configure them. This is not necessarily a great use case for terraform as a single change to the user_data would result in the instance being redeployed. For software deployment and desired state there are tools more suited such as ansible or puppet to name a few. Seeing as this is a Terraform demo we will stick with using Terraform but you should consider different tools for any configuration management.

module : security-groups
------------------------

We will use an existing module from the terraform registry for our security groups. The second module in the registry under the AWS provisioner is for security groups;

{{< figure src="image-1.png" >}}

security-group module

If we take a look at the provisioning instructions it implied we do not need a lot of configuration;

```HCL
module "security-group" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "3.18.0"
  # insert the 2 required variables here
}
```

We need a few more than the 2 required variables. We will be defining two SG, one for the proxy server and one for the Prometheus server. Here is our custom rules for the security groups with comments inline.

```HCL
module "security-group-prometheus" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "3.18.0"
  name = "SGAllowPrometheus"                         # Name of the Security group
  vpc_id = module.vpc.vpc_id                         # VPC Id using the value from the vpc module
  tags = merge(                                      # Tagging - add the tag variables and merge with name
    var.tags, {
      Name = "SgAllowPrometheus"
  })
  ingress_with_cidr_blocks = [                       # SG custom ingress rules for Prometheus server which is in 
    {                                                # a Private subnet
      from_port   = 9090
      to_port     = 9100
      protocol    = "tcp"
      cidr_blocks = "0.0.0.0/0"
    },
    {
      from_port   = 9182
      to_port     = 9182
      protocol    = "tcp"
      cidr_blocks = "0.0.0.0/0"
    },]
    egress_with_cidr_blocks = [                      # SG Egress rules for Prometheus server
    {
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = "0.0.0.0/0"
    },
  ]
}

module "security-group-proxy" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "3.18.0"
  name = "SgAllowProxy"                             # Name of the Security group
  vpc_id = module.vpc.vpc_id                        # VPC Id using the value from the vpc module
  tags = merge(                                     # Tagging - add the tag variables and merge with name
    var.tags, {
      Name = "SgAllowProxy"
  })
  ingress_cidr_blocks = ["0.0.0.0/0"]               # SG predefined ingress rules for Proxy server which is in 
  ingress_rules = ["http-80-tcp"]                   # a public subnet
  egress_with_cidr_blocks = [
    {
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = "0.0.0.0/0"
    },
  ]
}
```

Now we need to define our EC2 instance profile.

Module : instance_profile
--------------------------

This will be a module we create, so lets start off creating a folder under the modules folder for our configuration. We will call this one modules/instance_profile

next we can define our modules main.tf and add the terraform configuration block defining the minimum required version and the resources we need for the instance profile/role.

```HCL
terraform {
    required_version = ">=0.14.8"
}
                                   
resource "aws_iam_role" "demo-ssm-role" {               # Define our resource role for systems manager
  name = var.role_name                                  # the role name from a variables
  assume_role_policy = jsonencode({                     # use jsonencode to format the role policy to a string
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Sid    = ""
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
  tags = merge(
    var.tags, {
      Name = var.tag_name
  })
}

resource "aws_iam_instance_profile" "ec2_profile" {
  name = var.ip_name                                      # variable to define the instance profile name
  role = aws_iam_role.demo-ssm-role.name                  # which role is associated with the instance profile
}

resource "aws_iam_role_policy_attachment" "ssm-attach" {  # define which existing policies to attach to the the role
  role       = aws_iam_role.demo-ssm-role.name            # name of the role
  count      = length(var.policy_arns)                    # loop through each policy arn defined in the variable
  policy_arn = var.policy_arns[count.index]
}
```

Now lets define our variables.tf for the instance_profile module

```HCL
variable "ip_name" {
    type = string                # instance profile name
    default = ""
}
variable "role_name" {           # role name
    type = string
    default = ""
}
variable "tag_name" {            # tag name key value
    type = string
    default = ""
}
variable "policy_arns" {         # list of arns to attach to the role
  description = "arns to add to ec2 role"
  type        = list(any)
  default     = [""]
}
variable "tags" {                # tags to apply
  description = "resource tags"
  type        = map(any)
  default = {}
}
```

For this module we need to know the instance profile name so we can use it for our EC2 instances role attachment. To retrieve this value we will define an output value in the output.tf file within the module path;

```HCL
output "iam_instance_profile_name" {
    value = aws_iam_instance_profile.ec2_profile.name
}
```

Next we need to call this module from our root module defining the few required values we setup;

```HCL
module "instance_profile" {
  source      = "./modules/instance_profile"                             # where we created our configuration
  ip_name     = "demo_ec2_profile"                                       # instance profile name
  role_name   = "demo_ssm_role"                                          # role name
  policy_arns = ["arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"] # list of policy arn to attach
  tag_name    = "demo-ec2-ssm-policy"                                    # the tag key name value
  tags        = var.tags                                                 # pass the root module tag variable values
}
```

Security groups and instance profile defined.

Summary
-------

It's getting a bit long so lets wrap up for now. We have successfully defined our AWS Network components, security groups and instance profile ready for our instances.

In [part 4](/post/terraform-multi-cloud-demo-part-4-aws-instances/) we will continue to deploy our AWS EC2 instances

Disclaimer: You may incur costs if you follow along with the exercise. Use at your own risk!