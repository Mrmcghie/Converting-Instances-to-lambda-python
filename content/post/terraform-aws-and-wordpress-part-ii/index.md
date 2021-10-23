---
title: 'Terraform, AWS and WordPress – Part II'
date: Mon, 21 Sep 2020 16:27:29 +0000
draft: false
tags: ['apache', 'Apache', 'AWS', 'AWS', 'bash', 'Blog', 'certbot', 'MYSQL', 'shell', 'Terraform', 'Terraform', 'vsftpd', 'WordPress', 'WordPress']
author: "Dave"
toc: true
featured_image: "/image/title/terraform_aws_wordpress-2.png"
image: ["/image/information-1015297_1280.jpg"]
---

This is a continuation of [Terraform, AWS and WordPress - Part I](/post/terraform-aws-wordpress). Let look deeper under the covers and focus on what the bootstrap part of the deployment is doing.

user\_data
----------

Just as a refresher, the userdata defined on an AWS EC2 instance is used to run commands or scripts on the instance when deployed. As we are deploying an Amazon Linux EC2 instance the commands will reflect this Operating System.

### Setting the scene

So lets carve the user\_data into something we can read and work with and pop some remarks in.

This first part is all about updating the OS and installing and configuring most of the services required. This will then put us in a good position for deploying WordPress

```
#!/bin/bash
#First off, lets update the instance
sudo yum update -y
#Lets install all the required applications
#httpd24 - Apache http server 2.4
#php72 - scripting language
#mysql57-server - Open source relational database
#php72-mysqlnd - Add mysql db support to PHP
sudo yum install -y httpd24 php72 mysql57-server php72-mysqlnd
#start the mysql service
sudo service mysqld start
#start the apache server
sudo service httpd start
#enable both apache and mysql startup settings to on (start
sudo chkconfig httpd on
sudo chkconfig mysqld on
#add the default ec2-user to the apache group
sudo usermod -a -G apache ec2-user
#change ownership of the /var/www path to ec2-user as the owner, apache as the group
sudo chown -R ec2-user:apache /var/www
#install ssl support for apache
sudo yum install -y mod24\_ssl
#enable the extra packages for enterprise Linux repositories (EPEL) required for some later installs
sudo yum-config-manager --enable epel
# Now we are creating a virtual host in apache which we can use later when automatically registering our certificates
sudo echo "<VirtualHost \*:80>" >> /etc/httpd/conf/httpd.conf
sudo echo "ServerName davehart.co.uk" >> /etc/httpd/conf/httpd.conf
sudo echo "DocumentRoot /var/www/html" >> /etc/httpd/conf/httpd.conf
sudo echo "</VirtualHost>" >> /etc/httpd/conf/httpd.conf
```

That's the OS bits sorted, lets take a look at certbot before we move on

Certbot
-------

[_Certbot_](https://certbot.eff.org/) is a great open source piece of software that works with _[Let's Encrypt](https://letsencrypt.org/getting-started/)_ certificate authority (CA).

```
#download certbot
sudo wget https://dl.eff.org/certbot-auto
#enable permission on certbot
sudo chmod a+x certbot-auto
#Took a while to work out but the below line will agree to the terms of service, register a certificate with my email address using apache settings (it will use the http virtualhost we setup previously to reverse lookup and validate the domain. Last off it will restart the apache service
sudo ./certbot-auto --authenticator apache --debug --agree-tos -m "myemailaddress" --installer apache -d "davehart.co.uk" --pre-hook "httpd -k stop" --post-hook "httpd -k start" -n
```
{{< imgproc info Resize "100x" >}}
_I did get a bit carried away when testing my deployment. As such I hit a [rate limit](https://letsencrypt.org/docs/rate-limits/) with let's encrypt, ooppss. This stopped me registering any more certs with the domain I was using. It was only a test domain but good to know_ :smiley:

Lets see what has changed in the apache configuration at /etc/httpd/conf/httpd.conf file. If you remember from the previous block of code we created the following section

{{< figure src="image-25.png" >}}

How does it look now?

{{< figure src="/image/image-24.png" >}}

A few changes then! Lets also take a look at this new include file mentioned after the VirtualHost section

{{< figure src="/image/image-26.png" >}}

Cool. _Certbot_ has registered a certificate for us at _Let's Encrypt_ for the domain davehart.co.uk. Certificates have been downloaded and its also added a rewrite condition which does not do much that I can see. The rewrite rule does ensure that any http requests are rewritten as https. It then restarted the apache service so we are all secure, tidy :)

user\_data continued
--------------------

It's all coming along nicely. We now have a webserver with valid ssl certificates up and running, so what do all those other lines of user\_data do for us?

```
#download the latest version of WordPress
sudo wget https://wordpress.org/latest.tar.gz
#extract the tarball
sudo tar -xzf latest.tar.gz
#Now we are going to throw some mysql hardening lines in
mysql -u root -e "delete from mysql.user where user='';drop database if exists test;delete from mysql.db where db='test' or db='test\\\\\_%';flush privileges;"
#lets create a user for WordPress to use in mysql
mysql -u root -e "create user 'a-user-for-wordpress'@'localhost' identified by 'superstrongpassword';create database wordpressdb;grant all privileges on wordpressdb.\* to 'a-user-for-wordpress'@'localhost';FLUSH PRIVILEGES;"
#lets copy the sample php file so we have a backup
cp /wordpress/wp-config-sample.php /wordpress/wp-config.php
#Now we are going to use the stream editor to replace some of the configuration elements. Check the info block bellow for a brief overview of sed
sed -i "s/define( 'DB\_NAME', 'database\_name\_here' );/define( 'DB\_NAME', 'wordpressdb' );/g" /wordpress/wp-config.php
sed -i "s/define( 'DB\_USER', 'username\_here' );/define( 'DB\_USER', 'a-user-for-wordpress' );/g" /wordpress/wp-config.php
sed -i "s/define( 'DB\_PASSWORD', 'password\_here' );/define( 'DB\_PASSWORD', 'superstrongpassword' );/g" /wordpress/wp-config.php
sed -i "s/define( 'AUTH\_KEY',         'put your unique phrase here' );/define( 'AUTH\_KEY',         'uniquesalt' );/g" /wordpress/wp-config.php
sed -i "s/define( 'SECURE\_AUTH\_KEY',  'put your unique phrase here' );/define( 'SECURE\_AUTH\_KEY',  'uniquesalt' );/g" /wordpress/wp-config.php
sed -i "s/define( 'LOGGED\_IN\_KEY',    'put your unique phrase here' );/define( 'LOGGED\_IN\_KEY',    'uniquesalt' );/g" /wordpress/wp-config.php
sed -i "s/define( 'NONCE\_KEY',        'put your unique phrase here' );/define( 'NONCE\_KEY',        'uniquesalt' );/g" /wordpress/wp-config.php
sed -i "s/define( 'AUTH\_SALT',        'put your unique phrase here' );/define( 'AUTH\_SALT',        'uniquesalt' );/g" /wordpress/wp-config.php
sed -i "s/define( 'SECURE\_AUTH\_SALT', 'put your unique phrase here' );/define( 'SECURE\_AUTH\_SALT', 'uniquesalt' );/g" /wordpress/wp-config.php
sed -i "s/define( 'LOGGED\_IN\_SALT',   'put your unique phrase here' );/define( 'LOGGED\_IN\_SALT',   'uniquesalt' );/g" /wordpress/wp-config.php
sed -i "s/define( 'NONCE\_SALT',       'put your unique phrase here' );/define( 'NONCE\_SALT',       'uniquesalt' );/g" /wordpress/wp-config.php 
#copy all the wordpress file to the web root
cp -r wordpress/\* /var/www/html/
#oh no, more sed :| The below will allow the server to process any directives with the .htaccess context
sed -i '/<Directory "\\/var\\/www\\/html">/,/<\\/Directory>/ s/AllowOverride None/AllowOverride all/' httpd.conf
#install the GD Graphics library for php. Required for graphics manipulation
sudo yum install php72-gd -y
#restart apache
sudo service httpd restart
```

{{< imgproc info Resize "100x" >}}

sed : sed is short for stream editor. sed can edit line by line and requires no interaction, great! little example : _sed -i "s/define( 'DB\_NAME', 'database\_name\_here' );/define( 'DB\_NAME', 'wordpressdb' );/g" /wordpress/wp-config.php_

This line will search the file _**/wordpress/wp-config.ph**_ and will match the pattern _**"define( 'DB\_NAME', 'database\_name\_here' )"**_. When it finds a match it will replace it with _**"define( 'DB\_NAME', 'wordpressdb' )"**_

FTP
---

You will soon discover that you now have a WordPress server with a slight flaw. You cannot change themes, add widgets etc as you have no ftp access. This section will cover the FTP service install and configuration. Stay with me now, we are nearly there.

There are a load of ftp servers out there. Having no particular preference I came across [vsftpd](https://security.appspot.com/vsftpd.html) (first one google directed me at tbh). Lets see how we can install and configure this in our user\_data

user\_data discontinued...again
-------------------------------

```
#first off lets install vsftp
sudo yum install vsftpd -y
#sed is proving useful! Turn off anonymous access
sed -i "s/anonymous\_enable=Yes;/anonymous\_enable=No;/g" /etc/vsftpd/vsftpd.conf
#pasv settings - Passive File Transfer Protocol. Client initiates the data flow
sudo echo "pasv\_enable=YES" >> /etc/vsftpd/vsftpd.conf
sudo echo "pasv\_min\_port=1024" >> /etc/vsftpd/vsftpd.conf
sudo echo "pasv\_max\_port=1048" >> /etc/vsftpd/vsftpd.conf
#next two lines allow me to use the domain name and not define an IP
sudo echo "pasv\_addr\_resolve=YES" >> /etc/vsftpd/vsftpd.conf
sudo echo "pasv\_address=davehart.co.uk" >> /etc/vsftpd/vsftpd.conf
create a Linux user we will use for ftp access
sudo useradd wordpress-user
sudo echo -e "SuperSecurePassword2" | passwd wordpress
#add the user to the apache group
sudo usermod -a -G apache wordpress
#set folder permissions
sudo chmod 2775 /var/www
#find directories /var/www and if found execute the folder permissions
sudo find /var/www -type d -exec sudo chmod 2775 {} \\;
#find files in /var/www and if found update the file permissions
sudo find /var/www -type f -exec sudo chmod 0664 {} \\;
#enable vsftp startup settings to on
sudo chkconfig vsftp on
```

I do believe that covers this article. Surprised to say I really enjoyed writing this article and if in the process I learn something, mission accomplished. I hope you found it interested.

Link to github repo -->>[link](https://github.com/daveihart/demo-tf-wordpress-site)

Follow on [article](/post/terraform-aws-and-wordpress-part-ii/) about lessons learnt building this deployment

If you have any question or comments, please let me know.

Stay tuned.