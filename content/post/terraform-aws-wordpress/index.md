---
title: 'First!  Terraform, AWS and WordPress - Part I'
date: Sat, 19 Sep 2020 02:59:40 +0000
draft: false
tags: ['apache', 'Apache', 'AWS', 'AWS', 'bash', 'Blog', 'HCL', 'MYSQL', 'shell', 'Terraform', 'Terraform', 'WordPress', 'WordPress']
author: "Dave"
toc: true
featured_image: "/image/title/terraform_aws_wordpress-1.png"
---

Well here we go then, where to start? It seemed only fitting that the first post covers the fun I had setting up this WordPress server and site. I looked around at some of the options available, payed hosting etc and decided that "hey, lets learn something and share the experience at the same time". Rather than manually setting up and configure the server (easy), I decided to try automate it as much as possible.

Enter Terraform.... I started looking at Terraform around 3 months ago and loved it. HCL (HashiCorp Configuration Language) was easy to follow and rather intuitive. As with a lot of technologies I find it best to have a practical use-case to experiment with so I ported my AWS test environment (combination of CF templates and PowerShell) into Terraform. I will discuss this in a separate post but the end result was two Terraform workspaces, one for the network layer and another to stand-up my Jenkins server when it was required. Another factor behind all this was cost saving, I am paying for this myself so If I can provision and destroy easily its win win.

In my quest for complete automation I still need to understand how/if I can use some of the Terraform cloud variables as AWS EC2 user\_data values, maybe someone can suggest ideas here :wink:

Starting out
------------

I had decided on WordPress as the engine for this blog as it has been around a while now and is still recommended by a lot of people. I wanted the server to be hosted on my AWS account and I wanted to deploy as code. How hard could it be! oh and I wanted to keep costs down.

As my network layer was already in place I jumped straight into the code. Seeing as it should be version controlled and I wanted to leverage the remote execution capabilities of terraform.io my first step was to create a github repository to hold the code.

Creating a repository
---------------------

I already have an account on GitHub and numerous private repositories so I won't be going into that. Suffice to say it's pretty easy to setup an account.

My repositories all have a naming standard <language/app-usage/location-activity>. In this instances I defined tf-davehart.co.uk-wordpress as the repository name and created it with a blank README.md which will be updated in due course. Pointless having code out there with no description , at my age I will forget what it's for over time!

*   Use the + drop-down menu and select New repository

{{< imgproc image4 Fit "800x600" >}}

*   Give the new repository a memorable name
*   Add a meaningful description
*   Make it Public or Private
*   Initialize this repository with a README
*   For the sake of this example I also added a .gitignore with a Terraform template. I do not plan on initialising this locally but its just good practice.
*   Click the big green button

{{< imgproc image-211 Fit "800x600" >}}

Once the repository is created, copy the https link from the code dialog;

{{< imgproc image-34 Fit "800x600" >}}

Microsoft Visual Studio Code
----------------------------

For development work I use Windows 10 with Microsoft Visual Studio Code. Depending on the use case I will either develop locally or will use a WSL deployment of Ubuntu. For this project I will just stick to local. Next task is to clone the repository we just created on GitHub using git.

*   Navigate to a folder where you want to host your clone of the repository

```
git init
git clone https://github.com/daveihart/tf-davehart.co.uk-wordpress_.git
```

{{< imgproc image-52 Fit "800x600" >}}

Terraform
---------

Now there are many ways to setup your Terraform project. I like to separate out the key sections into their own files. For this project I started with three files. You can deploy your HCL all in one file but I like to think of the possibility to expand on a project and it would be easier if you do all the groundwork from the start. This method will also make it easier to trawl through the code in smaller files should you encounter any issues.

{{< imgproc image-61 Fit "600x400" >}}

The variables file will hold any variables required for the deployment. Hopefully the optional descriptions added will be enough

```
# VARIABLES

variable "region" {
  description = "AWS region to use when provisioning"
  type        = string
  default     = "eu-west-2"
}
variable "key_name" {
  description = "ec2 instance keypair to use when provisioning"
  type        = string
  default     = "supersecretkeypair"
}
variable "env_prefix" {
  description = "prefix used for tags and the like"
  type        = string
  default     = "dev"
}
variable "instance_size" {
  description = "instance type mapping based on role"
  type        = map(string)
  default     = { wordpress = "t2.micro" }

}
variable "dns_zone_id" {
  description = "zone id for route 53"
  type        = string
  default     = "supersecretdnszone"
}
variable "wordpress_count" {
  description = "number of wordpress servers to deploy"
  type        = number
  default     = 1
} 
```

The providers file will only need to hold the AWS provider for this project.

```
# PROVIDERS
provider "aws" {
  region = var.region
}
```

Now here is the big one, the WordPress file. This is where all the good stuff is carried out. I am going to break this down into it's component parts as it will be easier to explain. I will also publish this all as public GitHub repo for others to fork and play around with themselves. Best way to learn!

Lets start...

The **backend** is used to determine how state is loaded and how the plan and apply are executed. In this example I am using a **remote backend** which is Terraform cloud. My organisation has been defined and a workspace to hold the app state, runs, approvals etc.

```
# BACKEND

terraform {
  backend "remote" {
    organization = "my_organisation"
    workspaces {
      name = "tf_davehart_wordpress"
    }
  }
}
```

The **data sources** in this deployment are defining which VPC, subnet, and AMI we will be using. When deploying the network I use the same environment code. For example we are using a VPC with the name "dev-vpc", the subnet is the "dev-public" subnet and the ami is the latest aws-linux hvm ami using ebs type volumes.

```
# DATA Sources
data "aws_vpc" "vpc" {
  tags = {
    Name = "${var.env_prefix}-vpc"
  }
}
data "aws_subnet" "public" {
  tags = {
    Name = "${var.env_prefix}-public"
  }
}
data "aws_ami" "aws-linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn-ami-hvm*"]
  }
  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

Now we start defining our **resources**. First up, security groups.

The table reflects what we are looking to define

| Direction | protocol | port(s) | cidr | Usage |
| --------- | -------- | ------- | ---- | ----- |
| ingress | tcp | 22 | 0.0.0.0/0 | ssh access |
| ingress | tcp | 443 | 0.0.0.0/0 | HTTPS/SSL |
| ingress | tcp | 20-21 | 0.0.0.0/0 | ftp | 
| ingress | tcp | 1024-1048 | 0.0.0.0/0 | ftp | 
| ingress | tcp | 80 | 0.0.0.0/0 | HTTP (used for certbot) |
| egress | all | all | 0.0.0.0/0 | 

```
# RESOURCES

# SECURITY GROUPS #
# WordPress security group 
resource "aws_security_group" "wordpress-sg" {
  name   = "${var.env_prefix}_wordpress_sg"
  vpc_id = data.aws_vpc.vpc.id
  # SSH access from anywhere
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTPS access
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # FTP access
  ingress {
    from_port   = 20
    to_port     = 21
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # FTP access
  ingress {
    from_port   = 1024
    to_port     = 1048
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

# HTTP access - needed for Certbot
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # outbound internet access
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
} 
```

Next up we define our instance. This is where we attempt to minimise any configuration post deployment by defining a lot of user\_data (AWS bootstrap). I will dedicate a separate page to this and drill down into some of the other parts such as certbot and ftp. The eagled eyed will notice the odd password slipping into the user\_data lines. This is next on my list of things to remove. I would like to pass these in from Terraform cloud as sensitive environment variables.

```
# RESOURCES
...Continued
# INSTANCES #
resource "aws_instance" "wordpress" {
  count                  = var.wordpress_count
  ami                    = data.aws_ami.aws-linux.id
  instance_type          = var.instance_size["wordpress"]
  subnet_id              = data.aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.wordpress-sg.id]
  key_name               = var.key_name
  tags      = merge(local.common_tags, { Name = "${var.env_prefix}-wordpress" })
  user_data = <<EOF
          #!/bin/bash
          sudo yum update -y
          sudo yum install -y httpd24 php72 mysql57-server php72-mysqlnd
          sudo service mysqld start
          sudo service httpd start
          sudo chkconfig httpd on
          sudo chkconfig mysqld on
          sudo usermod -a -G apache ec2-user
          sudo chown -R ec2-user:apache /var/www
          sudo yum install -y mod24_ssl
          sudo yum-config-manager --enable epel
          sudo wget https://dl.eff.org/certbot-auto
          sudo chmod a+x certbot-auto
          sudo echo "<VirtualHost *:80>" >> /etc/httpd/conf/httpd.conf
          sudo echo "ServerName davehart.co.uk" >> /etc/httpd/conf/httpd.conf
          sudo echo "DocumentRoot /var/www/html" >> /etc/httpd/conf/httpd.conf
          sudo echo "</VirtualHost>" >> /etc/httpd/conf/httpd.conf
          sudo ./certbot-auto --authenticator apache --debug --agree-tos -m "myemailaddress" --installer apache -d "davehart.co.uk" --pre-hook "httpd -k stop" --post-hook "httpd -k start" -n
          sudo wget https://wordpress.org/latest.tar.gz
          sudo tar -xzf latest.tar.gz
          mysql -u root -e "delete from mysql.user where user='';drop database if exists test;delete from mysql.db where db='test' or db='test\\_%';flush privileges;"
          mysql -u root -e "create user 'a-user-for-wordpress'@'localhost' identified by 'superstrongpassword';create database wordpressdb;grant all privileges on wordpressdb.* to 'a-user-for-wordpress'@'localhost';FLUSH PRIVILEGES;"
          cp /wordpress/wp-config-sample.php /wordpress/wp-config.php
          sed -i "s/define( 'DB_NAME', 'database_name_here' );/define( 'DB_NAME', 'wordpressdb' );/g" /wordpress/wp-config.php
          sed -i "s/define( 'DB_USER', 'username_here' );/define( 'DB_USER', 'a-user-for-wordpress' );/g" /wordpress/wp-config.php
          sed -i "s/define( 'DB_PASSWORD', 'password_here' );/define( 'DB_PASSWORD', 'superstrongpassword' );/g" /wordpress/wp-config.php
          sed -i "s/define( 'AUTH_KEY',         'put your unique phrase here' );/define( 'AUTH_KEY',         'uniquesalt' );/g" /wordpress/wp-config.php
          sed -i "s/define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );/define( 'SECURE_AUTH_KEY',  'uniquesalt' );/g" /wordpress/wp-config.php
          sed -i "s/define( 'LOGGED_IN_KEY',    'put your unique phrase here' );/define( 'LOGGED_IN_KEY',    'uniquesalt' );/g" /wordpress/wp-config.php
          sed -i "s/define( 'NONCE_KEY',        'put your unique phrase here' );/define( 'NONCE_KEY',        'uniquesalt' );/g" /wordpress/wp-config.php
          sed -i "s/define( 'AUTH_SALT',        'put your unique phrase here' );/define( 'AUTH_SALT',        'uniquesalt' );/g" /wordpress/wp-config.php
          sed -i "s/define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );/define( 'SECURE_AUTH_SALT', 'uniquesalt' );/g" /wordpress/wp-config.php
          sed -i "s/define( 'LOGGED_IN_SALT',   'put your unique phrase here' );/define( 'LOGGED_IN_SALT',   'uniquesalt' );/g" /wordpress/wp-config.php
          sed -i "s/define( 'NONCE_SALT',       'put your unique phrase here' );/define( 'NONCE_SALT',       'uniquesalt' );/g" /wordpress/wp-config.php 
          cp -r wordpress/* /var/www/html/
          sed -i '/<Directory "\/var\/www\/html">/,/<\/Directory>/ s/AllowOverride None/AllowOverride all/' httpd.conf
          sudo yum install php72-gd -y
          sudo service httpd restart
          sudo yum install vsftpd -y
          sed -i "s/anonymous_enable=Yes;/anonymous_enable=No;/g" /etc/vsftpd/vsftpd.conf
          sudo echo "pasv_enable=YES" >> /etc/vsftpd/vsftpd.conf
          sudo echo "pasv_min_port=1024" >> /etc/vsftpd/vsftpd.conf
          sudo echo "pasv_max_port=1048" >> /etc/vsftpd/vsftpd.conf
          sudo echo "pasv_addr_resolve=YES" >> /etc/vsftpd/vsftpd.conf
          sudo echo "pasv_address=davehart.co.uk" >> /etc/vsftpd/vsftpd.conf
          sudo useradd wordpress-user
          sudo echo -e "7111065299081" | passwd wordpress 
          sudo usermod
          sudo chmod 2775 /var/www
          sudo find /var/www -type d -exec sudo chmod 2775 {} \;
          sudo find /var/www -type f -exec sudo chmod 0664 {} \;
    EOF
} 
```

Any finally we want our new instance to be associated with a DNS name. I recently registered the davehart.co.uk domain and as I wanted to be able to programmatically update the zone I changed the DNS NS (name Servers) to AWS, first defining the zone in Route 53.

Important point to note, we are defining a record (pointer) for the root (apex) of the domain. In AWS this has to be an A-record unless you are utilising services such as AWS ELB or AWS CloudFront. I might still consume some of the services but for this initial setup the A-Record works for me.

```
# Route53 #
resource "aws_route53_record" "wordpress" {
  zone_id         = var.dns_zone_id
  name            = "."
  type            = "A"
  ttl             = "5"
  records         = [aws_instance.wordpress[0].public_ip]
  allow_overwrite = true
} 
```

Before we commit this code I want to setup a workspace on my Terraform cloud and integrate this with the GitHub repository. When we do then push the code to GitHub it will trigger Terraform.

GitHub & Terraform cloud
------------------------

So we have already setup our GitHub repository, lets setup a new workspace on Terraform cloud to organise our infrastructure. I am not going to talk about setting up an account with Terraform cloud, just follow the [link](https://www.terraform.io/) and create an account.

Once logged into app.terraform.io, click

{{< imgproc image-71 Fit "800x600" >}}

Choose your workflow type, for this I am going to use the version control workspace. Your choice really

{{< imgproc image-81 Fit "800x600" >}}

I already have GiHub integrated with Terraform so I can just select that as my VCS. Steps [here](https://www.terraform.io/docs/cloud/vcs/github-app.html) if you are interested.

{{< imgproc image-100 Fit "800x600" >}}

Once I select GitHub it will list all of my repositories. I have selected the one relating to this project

{{< imgproc image-200 Fit "800x600" >}}

Once you select the Workspace you have a few options. Define the workspace name and in advanced options you can define your VCS branch, Terrform working dir, Workspace name, etc.

Click the button

{{< imgproc image-110 Fit "800x600" >}}

All goes well you will see a message like this,

{{< imgproc image-120 Fit "800x600" >}}

closely followed by

{{< imgproc image-130 Fit "800x600" >}}

For this deployment I created two environment variables (secret) to hold my AWS keys. The naming is important for the AWS provider to retrieve the correct values

{{< imgproc image-140 Fit "800x600" >}}

If we now jump back to our repository and look at the webhooks, we should see one for Terraform

{{< imgproc image-150 Fit "800x600" >}}

Integration complete.

Are we there yet?
-----------------

Now we have our HCL code all ready to go, our GitHub repo is primed to notify Terraform of any code changes, lets get our code commited and pushed to GitHub.

Back to Microsoft Visual Studio Code and in the PowerShell terminal window (You can have VS Code do all this for you but typing the commands helps me remember them :) )

```
git add *.tf
git commit -m "Initial"
git push
```

{{< imgproc image-160 Fit "800x600" >}}

Quick refresh on our GitHub repository

{{< imgproc image-170 Fit "800x600" >}}

Lets see what is happening over at Terraform...Nothing. I seem to have to kick off the first plan manually. Press

{{< imgproc image-180 Fit "800x600" >}}

Now this deployment gave me

{{< imgproc image-190 Fit "800x600" >}}

Good reason, I did not define valid environment variables as I have already deployed this and its where I am now writing this article :)

When all the variables are correct and you have a successful run, you will see

{{< imgproc image-210 Fit "800x600" >}}

Your Terraform plans and state are all in Terraform cloud. If you want to tear it all down, navigate to settings, destruction and deletion and queue a destroy plan

{{< imgproc image-220 Fit "800x600" >}}

I Hope you find this useful and maybe even considered dropping in again. Please take a look at [Part II](/post/terraform-aws-and-wordpress-part-ii/) which will dig into some of the user\_data and certbot. Great tool to allow you to automate external certificate registration and deployment.