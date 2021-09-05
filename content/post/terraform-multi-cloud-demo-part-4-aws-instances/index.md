---
title: 'Terraform: Multi-cloud demo – Part 4 – AWS Instances'
date: Fri, 09 Apr 2021 14:49:11 +0000
draft: true
tags: ['AWS', 'AWS', 'Blog', 'demo', 'GitHub', 'Remote State', 'remote-state', 'Terraform', 'Terraform']
author: "Dave"
toc: true
featured_image: "image/title/terraform-training-demo4-aws-instance.png"
---

Now we have our security groups and instance profile, lets look at creating our ec2 instances. We will be defining our own module which fits our requirements and introducing userdata and templatefile functions.

First we will setup and configure our modules main.tf (modules/instances/main.tf)

Module : instances
------------------

```
terraform {
    required\_version = ">=0.14.8"
}

resource "aws\_instance" "instance" {                                    # aws\_instance resource type
    ami                    = var.ami                                    # the ami id
    iam\_instance\_profile   = var.iam\_inst\_prof                          # the instance profile/role to attach
    instance\_type          = var.ec2\_instance\_size                      # the instance size
    subnet\_id              = var.ec2\_subnet                             # which subnet to deploy into
    vpc\_security\_group\_ids = var.vpc\_sg\_ids                             # security groups
    key\_name               = var.key\_name                               # ssh key
    tags                   = merge(var.ec2\_tags,{Name = var.ec2\_name})  # tags
    user\_data              = var.userdata                               # userdata to apply
}
```

Looks fairly simple. Lets declare our variables for the module;

```
variable "ec2\_instance\_size" {
    type = string
    default = "t2.micro"
}
variable "key\_name" {
    type = string
}
variable "ec2\_name" {
    type = string
}
variable "ec2\_tags" {
  description = "resource tags"
  type        = map(any)
  default =   {}
}
variable "iam\_inst\_prof" {
  description = "IAM Instance profile name"
  type        = string
  default     = ""
}
variable "userdata" {
  description = "user\_data template by type"
  type        = string
  default     = ""
}
variable "ec2\_subnet" {
  description = "user\_data template by type"
  type        = string
  default     = ""
}
variable "vpc\_sg\_ids" {
    description = "list of security groups"
    type = list
    default = \[""\]
}
variable "ami" {
  description = "AWS ami to use for instance"
  type        = string
  default     = ""
}
```

We will add some outputs in this module to make it easier to connect and test afterwards.

```
output "public\_ip" {
  value = join("", aws\_instance.instance\[\*\].public\_ip)
}

output "private\_ip" {
    value = join("", aws\_instance.instance\[\*\].private\_ip)
}
```

![](https://davehart.co.uk/wp-content/uploads/2020/09/information-1015297_1280-1024x1024.jpg)

The use of the join function allows us to concatenate a number of elements. By joining an empty string "" with the instance values we can prevent the code from failing should there be no value to output. It will then just output an empty string. For example, any instances deployed in the private subnets would not have a public\_ip to output so would result in a failure when applied.

Now we have our instances module setup, lets call it to deploy first the proxy (NGINX) server

```
module "proxy\_server" {
  source            = "./modules/instances"
  count             = 1
  ec2\_name          = "${"core-proxy"}${format("%02d",count.index+1)}"
  key\_name          = var.key\_name                                     # we will need to define this in our root variables.tf
  ec2\_subnet        = module.vpc.public\_subnets\[0\]                     # retrieve the public sn from the vpc module
  vpc\_sg\_ids        = \[module.security-group-prometheus.this\_security\_group\_id, module.security-group-proxy.this\_security\_group\_id\]                                          # retrieve the security group id from the module
  ami               = data.aws\_ami.aws-linux2.id                       # we will need a data source to get the latest ami
  iam\_inst\_prof     = module.instance\_profile.iam\_instance\_profile\_name# profile name from our module
  ec2\_instance\_size = var.instance\_size\["proxy"\]                       # instance size map variable. Need to define 
  ec2\_tags          = var.tags
  userdata          = templatefile("./templates/proxy.tpl",{proxy\_server = ("core-proxy${format("%02d",count.index+1)}"), prom\_server = module.prometheus\_server\[count.index\].private\_ip})
}

```

Looks a bit more complicated this time around. Recent changes to Terraform allowed the count to be used when calling a module which makes it easier for us to define reusable code. It is now similar to how you would execute a count within a resource.

Lets drill into some of the lines of code (I have added comments to the ones which are pretty self explanatory);

```
count             = 1
```

when you deploy a resource it will by default only configure one object. By adding the count meta-argument you can deploy multiple similar configurations. An alternative to using count would be for\_each but this requires the for\_each values are known before terraform performs any remote resource actions. As I am using the output of one instance module to set a value on another instance module (in the userdata) I am creating two modules but will be consuming the same source files.

One module could be used with for\_each but I would have to use another mechanism to configure the proxy server. Count is not necessary in this deployment but it is useful to show different ways you can utilise the count.index.

```
ec2\_name          = "${"core-proxy"}${format("%02d",count.index+1)}"
```

The ec2\_name value is appending the first interpolation of the servername with the count index (+1 as it starts at 0) and ensuring it is 2 decimal places. If we had a count of 2 we would end up with;

core-proxy01 , core-proxy02 and core-proxy03 as we iterate through the count

```
ec2\_instance\_size = var.instance\_size\["proxy"\]
```

The ec2\_instance\_size is referencing a variable which will be of type map. Depending what key is called (In this case it is "proxy") the value associated with the key will be returned. Lets add the variable to our root module variables.tf. I am using t2.micro for both instances to reduce cost but I am sure you get the idea :)

```
variable "instance\_size" {
  description = "instance type mapped to role"
  type        = map(any)
  default = {
    prometheus = "t2.micro"
    proxy      = "t2.micro"
  }
}
```

Next we will define which ami to use for our instances. I am looking to use the latest amazon linux2 ami

```
ami               = data.aws\_ami.aws-linux2.id 
```

This line is referring to a data source names aws\_ami which we have not yet defined. Lets define this in our root module main.tf adding some comments inline.

```
data "aws\_ami" "aws-linux2" {
  most\_recent = true                                            # Retrieve the latest ami
  owners      = \["amazon"\]                                      # It must be owned by amazon
  filter {                                                      # add some filters
    name   = "name"                                     
    values = \["amzn2-ami-hvm\*"\]                                 # The name must begin with amzn2-ami-hvm
  }
  filter {
    name   = "root-device-type"
    values = \["ebs"\]                                            # The root volume must be of type ebs
  }
}
```

Lastly, lets take a look at how we are providing the userdata using a templatefile.

```
userdata          = templatefile("./templates/proxy.tpl",{proxy\_server = ("core-proxy${format("%02d",count.index+1)}"), prom\_server = module.prometheus\_server\[count.index\].private\_ip})
}
```

![](https://davehart.co.uk/wp-content/uploads/2020/09/information-1015297_1280-1024x1024.jpg)

The [templatefile](https://www.terraform.io/docs/language/functions/templatefile.html) reads the file at the given path and renders its content as a template using a supplied set of template variables.

We have mentioned a template file in the templates folder named proxy.tpl. In our example this is a shell script which will be looking for a proxy\_server and prom\_server variable for it to execute. You can see that this script is now relying on the completion of a prometheus\_server module which we will define shortly.

Below is a snippet of the code deployed in the templates/proxy.tpl (The complete file will be in the git repository)

```
#!/bin/sh
server\_name="${proxy\_server}"                               # Here is the proxy\_server variable we are passing in
sudo hostnamectl set-hostname $server\_name --static
sudo yum update -y
sudo amazon-linux-extras install nginx1 -y
sudo rm -f /etc/nginx/nginx.conf
sudo cat > /etc/nginx/nginx.conf <<EOF
user nginx;
worker\_processes auto;
error\_log /var/log/nginx/error.log;
pid /run/nginx.pid;
include /usr/share/nginx/modules/\*.conf;
events {
    worker\_connections 1024;
}
http {
    server {
        listen       80;
        listen       \[::\]:80;
        server\_name  \_;
        location /prometheus/ {
        proxy\_pass http://${prom\_server}:9090;            # Here is the prom\_server variable we are passing in
        }
        # Load configuration files for the default server block.
        include /etc/nginx/default.d/\*.conf;
        error\_page 404 /404.html;
            location = /40x.html {
        }

        error\_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
}
EOF
.....
```

We will now add the Prometheus servers module to the root module main.tf using the same source.

```
module "prometheus\_server" {
  source            = "./modules/instances"
  count             = 1
  ec2\_name          = "${"core-prom"}${format("%02d",count.index+1)}"
  key\_name          = var.key\_name
  ec2\_subnet        = module.vpc.private\_subnets\[0\]
  vpc\_sg\_ids        = \[module.security-group-prometheus.this\_security\_group\_id, module.security-group-proxy.this\_security\_group\_id\]
  ami               = data.aws\_ami.aws-linux2.id
  iam\_inst\_prof     = module.instance\_profile.iam\_instance\_profile\_name
  ec2\_instance\_size = var.instance\_size\["prometheus"\]
  ec2\_tags          = var.tags
  userdata          = templatefile("./templates/prometheus.tpl", {prometheus\_server = ("core-prom${format("%02d",count.index+1)}")})
}
```

Snippet of the prometheus.tpl shell script which is only expecting the variable prometheus\_server;

```
#!/bin/bash
#Update and Install
servername="${prometheus\_server}"                       # Here is the prometheus\_server variable we are passing in
sudo echo "127.0.0.1   "$servername >> /etc/hosts
sudo hostnamectl set-hostname $servername --static
sudo yum update -y
sudo yum install wget -y
sudo useradd --no-create-home svc-prom
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.19.0/prometheus-2.19.0.linux-amd64.tar.gz
tar xvfz prometheus-2.19.0.linux-amd64.tar.gz
....

```

That should be all the code we need to write. Lets try it out!

### Terraform init

We will first initialise our Terraform configuration so Terraform can prepare our working directory for its use.

![](https://davehart.co.uk/wp-content/uploads/2021/04/demo4-terraform-init.gif)

Terraform output for the init phase

All good, lets run the plan

![](https://davehart.co.uk/wp-content/uploads/2021/04/demo4-terraform-plan.gif)

Terraform output for the plan phase

No errors and a quick look through the plan shows the expected resources being deployed. Lets apply our demo plan!

![](https://davehart.co.uk/wp-content/uploads/2021/04/demo4-terraform-apply.gif)

terraform apply demo.tfplan

Lets take a look in our aws console and see what we have.

![](https://davehart.co.uk/wp-content/uploads/2021/04/demo4-aws-console-network.gif)

Source control
--------------

All of the code used in this demo is available in my [git repository](https://github.com/daveihart/tf-training-demo1)

Summary
-------

That ends the demonstration. Through the 4 parts we have setup our source control, deployed azure infrastructure to host our Terraform remote state file using terraform and lastly we have deployed numerous AWS infrastructure components including a NGINX proxy server and Prometheus server configured completely using the userdata (bootstrap).

If you followed along and with to tear down your infrastructure it should be pretty simple.

1.  Move you state back to local by changing the root module backend back to local and then run terraform init

```
terraform {
    backend "local" {
    }
}
```

![](https://davehart.co.uk/wp-content/uploads/2021/04/image-2.png)

Moving state back to local with terraform init

2\. Once the state is back local, run a terraform destroy

![](https://davehart.co.uk/wp-content/uploads/2021/04/image-3.png)

Output from terraform destroy

![](https://davehart.co.uk/wp-content/uploads/2021/04/image-4.png)

Feel free to comment or contact me with any queries. Happy terraforming

Disclaimer: You may incur costs if you follow along with the exercise. Use at your own risk!