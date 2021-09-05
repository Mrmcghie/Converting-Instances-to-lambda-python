---
title: 'Automation: Apache and Route 53'
date: Fri, 15 Jan 2021 16:04:10 +0000
draft: false
tags: ['Amazon Linux', 'apache', 'Apache', 'AWS', 'AWS', 'Blog', 'cli', 'cli', 'GitHub', 'linux', 'route53', 'shell', 'shellscript']
author: "Dave"
toc: true
featured_image: "image/apache_route53_proxy_automation.png"
---
This solution consisted of a Route53 hosted zone with A-records directing traffic to an AWS EIP (Elastic IP Address) hosted on a firewall appliance (Fortinet). The firewall had a VIP rule to forward requests received on that EIP to the Apache reverse proxy.

A regular activity I had to perform for one of my clients was setting up apache reverse proxy services for their development environments. This involved defining a subdomain name for an application, creating an A-record on AWS Route53 and then configuring the rewrite rules which varied per application type. Lets look at how it was automated.

Method
------

To achieve the automation flow we needed to understand each component in the process and how we could automate it. First up was the AWS Route53 hosted zone, this should be easy using the AWS cli. Second was the firewall, as the method used would create a new child domain to the existing domain the current firewall VIP rule was fine, no change there. Lastly was the RHEL Apache server which could be updated with some bash code by adding a new rewrite rule. All of this could be kicked off from a [gocd](https://www.gocd.org/) pipeline with a few variables.

Considerations
--------------

As the code would be interacting with AWS resources (Route53 in this case), it would be advisable to introduce an IAM role to grant the server access to update the Route53 rules. The gocd agent server could be used by multiple teams so I decided to assign the roles to the Apache proxy server which was in my teams support alongside the AWS infrastructure. This did mean I needed to workout how to pass environment variables from the gocd server to the apache server. Once the variables needed by the code were available on the apache server the processing could all be performed there.

Process Flow
------------

This is what the processing would look like

{{< figure src="image2.png" >}}

Setting the scene
-----------------

A number of activities had to be performed to prepare the environment for this type of automation activity. The apache server was configured to use a single httpd.conf file with all the rules within. This would be a pain to append a new rule each time. As such I changed the server to use individual rewrite rule conf files located in the /etc/httpd/conf.d folder. Each rewrite rule had its own file. This made the process more granular and if and when I made a remove rule process it would be less risky.

In addition to the above, I also had to define some limitations on the rewrite rules being created. There were three distinct application rewrite configurations so the process would have to include an application type variable. If it did not fit in the agreed application types then it would not execute. These three types could then be defined as templates within the code and would also used in working out the child domain name and suffix (numeric value)

AWS Roles
---------

A role was created to allow the apache server access to manage AWS Route 53. This was attached to the ec2 instance hosting the apache service.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "route53:ListTrafficPolicyInstances",
                "route53:GetTrafficPolicyInstanceCount",
                "route53:GetChange",
                "route53:ListTrafficPolicyVersions",
                "route53:TestDNSAnswer",
                "route53:GetHostedZone",
                "route53:GetHealthCheck",
                "route53:ListHostedZonesByName",
                "route53:ListQueryLoggingConfigs",
                "route53:GetCheckerIpRanges",
                "route53:ListTrafficPolicies",
                "route53:ListResourceRecordSets",
                "route53:ListGeoLocations",
                "route53:GetTrafficPolicyInstance",
                "route53:GetHostedZoneCount",
                "route53:GetHealthCheckCount",
                "route53:GetQueryLoggingConfig",
                "route53:ListReusableDelegationSets",
                "route53:GetHealthCheckLastFailureReason",
                "route53:GetHealthCheckStatus",
                "route53:ListTrafficPolicyInstancesByHostedZone",
                "route53:ListHostedZones",
                "route53:ListVPCAssociationAuthorizations",
                "route53:GetReusableDelegationSetLimit",
                "route53:ChangeResourceRecordSets",
                "route53:GetReusableDelegationSet",
                "route53:ListTagsForResource",
                "route53:ListTagsForResources",
                "route53:GetAccountLimit",
                "route53:ListTrafficPolicyInstancesByPolicy",
                "route53:ListHealthChecks",
                "route53:GetGeoLocation",
                "route53:GetHostedZoneLimit",
                "route53:GetTrafficPolicy"
            ],
            "Resource": "*"
        }
    ]
}
```

Authorised user
---------------

I defined a new key pair on the apache server for the automation activity which could be passed to the deployment teams for their pipelines.

The new key pair was generated using ssh-keygen. The public key generated was then added to the machines authorized\_keys file and the private key used in the pipeline.

Generate a public and private key pair.

{{< figure src="image-21.png" >}}

Once we have our key pair we need to copy the public part (filename.pub) into the ~/.ssh/authorized_keys file

{{< figure src="image-31.png" >}}

{{< figure src="image-41.png" >}}

In the above screenshot you can see the default public key for the user and the newly generated example key. The private part of the key can now be passed to the deployment team for them to use in their pipeline.

You will need to ensure that the automation account has sufficient file level permissions to update apache and store any backup etc. generated by the script.

Passing environment variables
-----------------------------

As mentioned I wanted to pass variables from the gocd pipeline server to the apache server. Reason being that any variables defined in the pipeline would create environment variables on the gocd agent. This would then pass them over to the apache server at execution. The script would then execute on the apache server using these values. After a bit of head scratching I came across SendEnv which did this very thing!

### SendEnv

SendEnv is one of the many ssh options. The manual defines it as ;

{{< figure src="image-60.png" >}}

That makes it so clear!

So to set this up I had to update the sending server (gocd agent) /etc/ssh/ssh\_config and the receiving server (apache) /etc/ssh/sshd_config

/etc/ssh/ssh_config

```
SendEnv var1 var2 var3
```

/etc/ssh/sshd_config

```
AcceptEnv var1 var2 var3
```

So any environment variables defined in the pipeline following the names could be passed across

{{< figure src="image-70.png" >}}

And then the task configuration

{{< figure src="image-80.png" >}}

All environment variables prefixed with DEMO_URL_ will be passed to the ssh target

Script
------

So lets take a look at the shell script. Similar to my previous scripts we break it down into functions. I will also include remarks to elaborate on some of the code. Not everyone will want or need the remarks so there will be a link at the end to the public GitHub repo.

### Declarations

This is where we reassign the environment variables sent from the gocd agent using SendEnv. We also perform some concatenation of values which we will use later in the script. Note that my environment was pretty static so I did hardcode a number of the AWS parameters used for Route 53. All of these could be defined as pipeline variables.

```bash
#!/bin/bash
#Local declarations based on Env variables passed from gocd agent server
sudo application=${DEMO_URL_APPLICATION,,}
envcode=${DEMO_URL_ENVCODE,,}
internalhostname=${DEMO_URL_INTERNALHOSTNAME,,}
internalport=$DEMO_URL_INTERNALPORT
type="https"
##### Calculated variables####################
ProxyPass=$type":"$internalhostname":"$internalport"/"
ProxyPassReverse=$type":"$internalhostname":"$internalport"/"
#####   start - static variables    #####
supp\_apps=("first_app" "second_app" "third_app") # Used to validate supported rule is being defined
dt=`date +%F"-"%H"-"%M`
# Details for AWS Route53
ip="0.0.0.0" # External IP used for Route53 to redirect traffic
type="A"
ttl=300
hostedzoneid="XXXXXXXXXXXXXXXXX" # Your AWS Hosted Zone id
dnssuffix=".example.com"    #suffix to add to entry
comment="Programmatically created DNS Entry for "$record" created on "$dt
checkpath="/etc/httpd/conf.d/"
initialrecord=$envcode$application

```

Functions
---------

### validate_app

This function ensures the inputs (variables) provided are valid so they process can successfully finish. It first checks if the application type is valid as it relies on this to define which rewrite template to use. Secondly it check to ensure the port is valid as far as it being numeric. You could validate this further by defining an allowed port range. Next it will check if there is an environment code and lastly has an internal hostname be provided. All of these inputs are required to deploy the configuration.

```bash
function validate_app
{
    echo "Checking if application name "$application" is valid"
    if ( IFS=$'n'; echo "${supp_apps[*]}" ) | grep -qFx "$application"; then
        echo "found application "$application" in approved list, continuing"
    else
        echo "error : "$application" was not found in the list of approved applications"
        echo "Approved applications are : "
        for eachapp in "${supp_apps[@]}"
            do
                echo $eachapp
            done
        echo "error: Please provide an approved application! exiting..."
        exit 1
    fi
	echo "Validating port is a number"
		ncheck='^\[0-9\]+$'
		if ! [[ $internalport =~ $ncheck ]] ; then
			echo "error : port provided is not a number! exiting..."
			exit 1
		else
			echo "Port is a number, continuing"
		fi
	echo "Checking that an environment has been provided"
		if [ -z "$envcode" ] ; then
			echo "error: No environment code has been provided! exiting..."
			exit 1
		else
			echo "Environment code has been provided, continuing"
		fi
	echo "Checking that an internal hostname has been provided"
		if [ -z "$internalhostname" ] ; then
			echo "error: No internal hostname has been provided! exiting..."
			exit 1
		else
			echo "Internal hostname has been provided, continuing"
		fi
}
```

### build_record

As each environment might have multiple deployments of an application, it was necessary to check what was already configured and then programmatically define the next value. This could have been an environment variable but this method removed possible human error.

It works by concatenating the environment name and application type and then starting at 01 it checks if this exists. If there is already a record it increments by 1 until it cannot find an existing record.

The checks are performed against an array which is populated with all of the record sets deployed in the AWS Route53 hosted zone.

As an example if could build out the name;

z1first_app it would add 01 to the end and check for a record z1first_app01. If this does not already exist the value will be used to build out the FQDN otherwise it will increment by one until it finds a free value. This occurs a max. 20 times.

```bash
function build_record
{
    #build the A record based on the provided variables calculating the next available address
    found=""
    rec_count=0
    existing_rec_array=()
    echo "Records to search Route53 : $initialrecord"
    all_records=(`aws route53 list-resource-record-sets --hosted-zone-id $hostedzoneid --output text --query "ResourceRecordSets[*].Name"`)
    echo "Route53 has ${#all_records[@]} registered A Records for $dnssuffix"
    if [ ${#all_records[@]} -gt 0 ]; then
        #echo "Identifying existing records with similar prefix for processing"
        for record in "${all_records[@]}"
        do
		#echo $record" is being checked!"
            if [[ "$record" == "$initialrecord" ]]; then
                #echo $record" already exists"
                existing_rec_array=($record)
                found="yes"
                rec_count=$(($rec_count+1))
            fi
        done
    fi
	#echo "Found variable set as "$found
    if [ "$found" = "" ]; then
        #echo "No records found for $initialrecord"
        record=$initialrecord"01"
        checkurl=$record$dnssuffix
	echo "DNS A Record will be created for "$record
    else
		# There are records following this name, step through and work out available suffix
        available=""
        available_count=1
        while [[ "$available" != "true" && "$available_count" -le 20 ]];
            do
                printf -v pad_available_count "%02d" $available_count
                #build the string to check for
                checkurl=$initialrecord$pad_available_count$dnssuffix
                checkurlprefix=$initialrecord$pad_available_count
                checkurldot=$checkurl"."
                echo "Checking if "$checkurl" is available"
                if ( IFS=$'\n'; echo "${existing_rec_array[*]}" ) | grep -qFx "$checkurldot"; then
                    echo "found existing A record for "$checkurl" , skipping"
                else
                    echo "Did not find any existing A record for "$checkurl
                    echo "We can proceed using "$checkurl" for the new A record"
                    record=$checkurlprefix
                    available="true"
                fi
                available_count=$(($available_count+1))
            done
    fi
}
```

### check_conf_d

This is a secondary check to ensure there are not any orphaned apache configurations. It is possible a free record could be calculated for Route53 and then an existing configuration file exists. Worth a manual check then.

If an existing conf file is located the pipeline exits.

```bash
function check_conf_d
{
    #ensure there are no existing configuration files with this name. Existing suggests an orphaned file as the names should match a route53 rule
    checkfile=$checkpath$checkurlprefix".conf"
    echo "Checking for existing conf file "$checkfile
    if [ -f "$checkfile" ]; then
        echo "There is already a config file for this rule. Please advise team to check for orphaned configuration files. Exiting!"
        exit 1
    else
        echo "No existing conf files for "$checkurlprefix" , continuing"
    fi
}
```

### update_route53

When using the AWS cli to add a Route53 A record it uses a JSON-formatted configuration file to provide the required configuration. With this requirement the first part of this function created a temporary .json file and populates it with values from either the environment variables or calculated values from previous steps.

Next the function will backup the existing Route53 records and then use the temporary .json file to create the new A-record. Finally the temporary .json file is removed

```bash
function update\_route53
{
    # First generate a json file with required details as Route53 cli uses this as the input stream
    TMPFILE=$(mktemp testXXX.json)
    echo "Temporary file created : "$TMPFILE
    cat > ${TMPFILE} << EOF
{
    "Comment":"$comment",
    "Changes":\[
    {
        "Action":"CREATE",
        "ResourceRecordSet":{
        "Name":"$record$dnssuffix",
        "Type":"$type",
        "TTL":$ttl,
        "ResourceRecords":\[{"Value":"$ip"}\]
        }
    }
    \]
}
EOF

#backup the current record sets first
backfile="/etc/httpd/route53/backup-"$dt".json"
echo "creating backup of existing Route53 records : "$backfile
aws route53 list-resource-record-sets --hosted-zone-id $hostedzoneid > $backfile
#use the temporary file to create a record on Route53 using the AWS cli
aws route53 change-resource-record-sets --hosted-zone-id $hostedzoneid --change-batch file://$TMPFILE
#Remove the temporary file
rm -f $TMPFILE
}
```

### build_proxy_rule

This function will build out the .conf file used by the apache server for the rewrite rule. Different rules are defined depending on which application type is being processes. Additional rules can be added in this function if required. The output file is created in the path defined in the checkpath variable (/etc/httpd/conf.d/)

```bash
function build_proxy_rule
{
    echo "creating proxy rule for "$record
    #create a filename
    conf_file=$checkpath$record".conf"
    if [ "$application" == "first_app" ] ;then
        cat > ${conf_file} << EOF
<VirtualHost *:443>
ServerName $checkurl
SSLEngine on
SSLProxyEngine on
SSLProxyCheckPeerCN off
SSLProxyCheckPeerName off
SSLCertificateFile      "conf/example-wildcard.crt"
SSLCertificateKeyFile   "conf/example-wildcard.key"
SSLProtocol      all -SSLv3 -SSLv2 -TLSv1
SSLCipherSuite ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256
RewriteEngine On
ProxyPass / $ProxyPass
ProxyPassReverse / $ProxyPassReverse
</VirtualHost>
EOF
    elif [ "$application" == "second_app" ] ;then
        cat > ${conf_file} << EOF
<VirtualHost *:443>
ServerName $checkurl
SSLEngine on
SSLProxyEngine on
SSLProxyCheckPeerCN off
SSLProxyCheckPeerName off
SSLCertificateFile      "conf/example-wildcard.crt"
SSLCertificateKeyFile   "conf/example-wildcard.key"
SSLProtocol      all -SSLv3 -SSLv2 -TLSv1
SSLCipherSuite ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256
RewriteEngine On
ProxyPass / $ProxyPass
ProxyPassReverse / $ProxyPassReverse
</VirtualHost>
EOF
    elif
[ "$application" == "third_app" ] ;then
        cat > ${conf_file} << EOF
<VirtualHost *:443>
ServerName      $checkurl
SSLEngine on
SSLProxyEngine on
SSLProxyCheckPeerCN off
SSLProxyCheckPeerName off
SSLCertificateFile      "conf/example-wildcard.crt"
SSLCertificateKeyFile   "conf/example-wildcard.key"
SSLProtocol      all -SSLv3 -SSLv2 -TLSv1
SSLCipherSuite ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256
RewriteEngine On
RewriteCond %{REQUEST\_METHOD} ^(TRACE|TRACK)
RewriteRule .\* - \[F\]
RewriteCond %{HTTP\_HOST} !^$record\\.example\\.com \[NC\]
RewriteRule ^/(.\*)https://$record$dnssuffix/\\$1 \[L,R\]
ProxyPreserveHost on
ProxyPass / $ProxyPass
ProxyPassReverse / $ProxyPassReverse
</VirtualHost>
EOF
    else
        echo "No proxy build process defined yet for this application. We should never see this message unless the validation has failed!"
    fi
}
```

### restart_service

This function will restart the httpd service on the apache server. Apache servers only load their configuration when the service starts.

If you are working in a production environment you would be scheduling this to run out of hours either as a separate task in your pipeline or manually agreed in a downtime window. Mine is development so no issues running at pipeline execution.

```
function restart_service
{
    echo "Restarting the httpd service"
    sudo httpd -k restart
    echo "******************************************************************"
    echo ""
    echo "External URL created : "$checkurl
    echo ""
    echo "******************************************************************"
}
```

### Main

Not really a main function as such but just the order of function execution. Located at the bottom of the script.

```bash
validate_app
build_record
check_conf_d
update_route53
build_proxy_rule
restart_service
```

Conclusion
----------

So what have we achieved with this process. By defining a handful of variables in a pipelines we can calculate a subdomain name, backup AWS Route53 records, create a new A record, create an apache rewrite rule and restart the apache service.

Mission accomplished!

I enjoyed developing this process as I had to think outside the box for a number of requirements. This also meant I had to do a fair bit of infrastructure reconfiguring before it was viable as the apache server was using a single httpd.conf file.

As usual, here is a [link](https://github.com/daveihart/bash-aws-apache_rewrite_automation) to my GitHub repo containing the source code. Feel free to ask any questions or suggest any improvements.