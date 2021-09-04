---
title: 'EC2 inventory using AWS Lambda and Python'
date: Mon, 05 Oct 2020 07:25:24 +0000
draft: false
tags: ['AWS', 'AWS', 'AWS Lambda', 'Blog', 'lambda', 'python', 'python', 'S3']
author: "Dave"
toc: true
featured_image: "image/title/python-aws-lambda-ec2-report.png"
---

A long while back I wrote a PowerShell script to produce a CSV file of EC2 instances across multiple accounts. The original PowerShell script was running on a Windows server as a scheduled task, oh how we have moved on. About 6 months ago I rewrote the script in Python and then moved it over to AWS Lambda. This was my first really opportunity to use AWS Lambda to execute code in a (Misnomer alert!!) **serverless** compute service.

AWS Services
------------

The full process makes use of a number of AWS services.

*   AWS Lambda for the code execution
*   AWS S3 as storage for the output CSV

Output example
--------------

The report produces a CSV file similar to the below screenshot.

{{< figure src="image-27" >}}

Input
-----

The output CSV file is constructed using input variables provided as variables to AWS Lambda. Define the attributes and tags and accounts you wish to include in the report in the AWS Lambda console page.

#### Lambda Environment variables

To ensure the code can easily be re-used, all the key elements are variables. These can then be updated by anyone without having to get involved in the coding side. e.g. adding a new account or tag can easily be done by updating the comma separated list for the item within the Lambda environment variables.

| Key | Value |
| --- | ----- |
|arole | role name to assume |
| attributes | **(comma separated list)** e.g. OwnerId,InstanceId,InstanceType |
| tags | **(comma separated list)** e.g. Name,Project,Release,Environment,CostCentre |
| aws_accounts | **(comma separated list)** e.g. 1111111111111,2222222222222,3333333333333 |
| bucket_name | name of bucket e.g. reporting_test_bucket |
| output_path | folder within the bucket where the report will be held. E.g. ec2\_reports |

Flow Chart
----------

This is what the process flow looks like

{{< figure src="ec2-reporting-flow" >}}

Code construct
--------------

Lets take a look at the code before we move into how its all executed

### Imports

Import all of the required external libraries. The ones we are using are all already available in Lambda so no library additions are required.

```bash
import boto3
import csv
import os
import botocore
from datetime import date
```

### Variables

The in-code variables have been setup to read from AWS Lambda environment variables. The variables prefixed with os.environ are retrieving the environment variables from the server (the serverless one :smile: )

Any hard coded variables are there as I have not yet updated the code to process the options so they are not configurable at this point. For example the report\_format from my initial Python code handles "formatted" (used in this code) and also "RAW" which dumps all values from the instances. That will follow on at some point. It was requested by someone but I never really liked the output, messy.

```python
#temporary location for processing
output_file="/tmp/temp.csv"
output_path = os.environ['output_path']
report_format = "formatted"
accounts_list = os.environ['aws_accounts']
accounts = accounts_list.split(",")
arole = os.environ['arole']
values_list = os.environ['attributes']
formatting_values = values_list.split(",")
tags_list = os.environ['tags']
formatted_tags = tags_list.split(",")
bucket_name = os.environ['bucket_name']
dt=date.today()
dtstr=dt.strftime("%Y%m%d")
s3_output_file=output_path +"/"+"ec2_report_"+dtstr+".csv"
```

Functions
---------

As usual, lets add functions to allow for repeatable script blocks. Note that a number of the functions are similar to those used in the _Terminate EC2 instance by tag using Python_ post. Why re-invent the wheel!

assume\_roles
-------------

If the AWS account being processed is not the AWS account the script is being executed from, you will need to assume a role defined in that target account.

```python
def assume_roles(acc,accounts,arole):
    global acc_key
    global sec_key
    global sess_tok
    global client
    print(f"Initating assume role for account : {acc}")
    sts_conn = boto3.client('sts')
    tmp_arn = f"{acc}:role/{arole}"
    print(tmp_arn)
    response = sts_conn.assume_role(DurationSeconds=900,RoleArn=f"arn:aws:iam::{tmp_arn}",RoleSessionName='Test')
    acc_key = response['Credentials']['AccessKeyId']
    sec_key = response['Credentials']['SecretAccessKey']
    sess_tok = response['Credentials']['SessionToken']
    print(f"Access key = {acc_key}")
```

### get\_instances

No filtering being done this time around. We are retrieving the defined attributes from all instances in each AWS account listed.

```python
def get_instances(process_acc,filters=[]):
    reservations = {}
    try:
        reservations = ec2.describe_instances(
            Filters=filters
        )
    except botocore.exceptions.ClientError as e:
        print(e.response['Error']['Message'])
    instances = []
    for reservation in reservations.get('Reservations', []):
        for instance in reservation.get('Instances', []):
            instances.append(instance)
    return instances 
```

### formatted\_report

This function is pulling in the attributes and tags we requested as environment variables. We are then creating a list of attributes and a list of tags. This makes it easier to perform the next function which writes out the CSV file. The two lists will makeup the fieldnames value for defining the column header which will be used in the proceeding function

```python
def formatted_report(instances,attrib_set,tag_set):
    #Build two lists. One for attributes and one for tags. These will be used to define the column values for the csv
    max=int(len(formatting_values))
    m=0
    for i in formatting_values:
        attrib_set.append(i)
    for t in formatted_tags:
        tag_set.append(t)
    tag_set = list(set(tag_set))
    attrib_set = list(set(attrib_set))
    print(attrib_set)
```

### report\_writer

The report\_writer function handles the output. All of the following items are handled in this function

The file type, file name, what the header will be

*   File type
*   File name
*   fields to include
*   header row
*   Some of the retrieved values need a little nudging (post processing) before adding them

```python
def report_writer(processing_acc,instances,attrib_set,tag_set):
    with open(output_file, 'a') as csvfile:
        fieldnames = attrib_set + tag_set
        writer = csv.DictWriter(csvfile, fieldnames=fieldnames)        
        if processing_acc == 1:
            writer.writeheader()
        for instance in instances:
            row = {}
            max=int(len(attrib_set))
            m=0
            for i in attrib_set:
                m +=1
                n = m-1
                if i == "State": #Its a list so needs more processing
                    tmpstate=instance.get(i)
                    row[attrib_set[n]] = tmpstate['Name']
                elif i == "Placement":
                    tmpplacement=instance.get(i)
                    row[attrib_set[n]] = tmpplacement['AvailabilityZone']
                elif i == "OwnerId":
                    nicinf=instance.get('NetworkInterfaces')
                    for val in nicinf:
                        for val2, val3 in val.items():
                            if val2 == "OwnerId":
                                row[attrib_set[n]] = str("'" + val3) + "'"
                elif i == "CoreCount":
                    vtype=instance.get('CpuOptions')
                    for vcore,cores in vtype.items():
                        if vcore == "CoreCount":
                            row[attrib_set[n]] = cores
                elif i == "ThreadsPerCore":
                    vtype=instance.get('CpuOptions')
                    for vcore,cores in vtype.items():
                        if vcore == "ThreadsPerCore":
                            row[attrib_set[n]] = cores
                else:
                    row[attrib_set[n]] = instance.get(i)
            for tag in instance.get('Tags', []):
                if tag.get('Key') in formatted_tags:
                    row[tag.get('Key')] = tag.get('Value')
            writer.writerow(row)
```

### lambda\_handler

The lambda\_handler is what AWS Lambda invokes when the script is executed. This is the main function which when called will invoke the preceding functions.

```python
def lambda_handler(event, context):
    global ec2
    global attrib_set
    global tag_set
    global instances
    global session
    global client
    global processing_acc
    global headers
    global writer
    attrib_set = []
    tag_set = []  
    processing_acc = 0
    headers = 0
    
    #get account this is executing in so we know whether to assume a role in another account
    client = boto3.client("sts")
    account_id = client.get_caller_identity()["Account"]    
    #define s3
    s3=boto3.client("s3")
    #remove the old output file if exists
    print(f"Checking for file : {output_file}")
    for acc in accounts:
        processing_acc += 1
        print(f"Processing account : {processing_acc}")
        if acc != account_id:
            assume_roles(acc,accounts,arole)
            ec2 = boto3.client('ec2',aws_access_key_id=acc_key,aws_secret_access_key=sec_key,aws_session_token=sess_tok,region_name='eu-west-1')
            instances = get_instances(processing_acc)
            if processing_acc == 1:
                formatted_report(instances,attrib_set,tag_set)
            report_writer(processing_acc,instances,attrib_set,tag_set)
        else:
            print(f"account {acc}")
            ec2=boto3.client('ec2')
            instances = get_instances(processing_acc)
            if processing_acc == 1:
                formatted_report(instances,attrib_set,tag_set)
            report_writer(processing_acc,instances,attrib_set,tag_set)
            #upload to s3
    print(F"uploading {s3_output_file} to s3")
    s3.upload_file(output_file, bucket_name, s3_output_file)    
    print("finished")
```

Security
--------

### Script execution

For each of the AWS accounts that are assuming the role, the role will need to exist in that account. The same role name must be used in each AWS account you are querying. The role must trust the account which the assume is called and have at a minimum Read Only EC2 access.

You can use the AWS Managed policy "AmazonEC2ReadOnlyAccess" to allow describing the EC2 instances.

Create a role which can be used by Lambda as the execution role. The role applied to Lambda must at a minimum have;

#### [](https://github.com/daveihart/python-aws-lambda-ec2_reporter/blob/master/README.md#assume-role-across-accounts)Assume Role across accounts

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "<strong>Resource</strong>": "arn:aws:iam::account_number:role/role_name"
        }
    ]
}
```

_The **Resource** listed in the above policy can be made into a list to define multiple accounts_[](https://github.com/daveihart/python-aws-lambda-ec2_reporter/blob/master/README.md#for-logging-to-cloudwatch-useful-for-audit-and-troubleshooting)

#### For logging to cloudWatch (useful for audit and troubleshooting)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```

#### [](https://github.com/daveihart/python-aws-lambda-ec2_reporter/blob/master/README.md#ability-to-upload-to-s3-bucket)Ability to upload to S3 bucket

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListAllMyBuckets",
                "s3:GetBucketLocation"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "s3:PutObject",
            "Resource": [
                "arn:aws:s3:::bucket_name",
                "arn:aws:s3:::bucket_name/*"
            ]
        }
    ]
}
```

### [](https://github.com/daveihart/python-aws-lambda-ec2_reporter/blob/master/README.md#s3-considerations)S3 access

Define a policy to allow the AWS Lambda function access to your S3 bucket

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListAllMyBuckets",
                "s3:GetBucketLocation"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "s3:PutObject",
            "Resource": [
                "arn:aws:s3:::BUCKETNAME",
                "arn:aws:s3:::BUCKETNAME/*"
            ]
        }
    ]
}
```

### S3 considerations

Unless you have any contractual or regulatory requirements to retain x amount of logs, you could introduce a Lifecycle rule to the s3 bucket to remove objects older than x days.

Trigger : Amazon EventBridge event
----------------------------------

Please use the following steps to define a schedule trigger to generate the log files

1.  On the AWS Console, navigate to CloudWatch Amazon EventBridge. Then select the Events | Rules
2.  Select the Create Rule button
3.  Select the schedule radio button and define your schedule. We used a cron expression 00 03 ? * * * This executed the code at 3am everyday of the week
4.  Select the add target button
5.  Choose Lambda function and then select your function from the drop down list
6.  Select the Configure details button
7.  Give the rule a meaningful name and description
8.  Select the Create rule button

#### Testing

To test your AWS Lambda function without using the event trigger, configure a test event and have nothing define in the json, e.g.

```
{}
```

#### AWS Lambda settings

This one always pops up when I develop code to run on AWS Lambda

{{< figure src="image-28.png" >}}

The key entry in that above screenshot is **Timeout** (actually they are all important this is just the one which will catch you out). The default value is 3 seconds. Consider that this function is describing instances across multiple accounts. The amount of time it takes is going to depend on many factors such as geographical locations, amount of instances to retrieve, number of accounts to gather from. That's just a few off the top of my head, I am sure there are more. The maximum timeout is 900 seconds so you will need to split the functions if this is not sufficient.

Planned enhancements
--------------------

Send the output as an email attachment. Under test.

Conclusion
----------

So lets sum up. We have deployed a python script into AWS Lambda. Granted AWS Lambda access to S3, AWS CloudWatch and AWS STS so it can gain temporary credentials and assume roles across accounts. We have defined roles in other AWS accounts which allow the account which is executing the AWS Lambda function access to describe EC2 instances. We have then generated a csv file with specific values we requested and posted this to an S3 bucket.

Great stuff. As usual, feel free to ask any questions or let me know if you have any suggestions on how to improve the process. AWS is a constantly evolving landscape so change is inevitable. Oh almost forgot, link to the GitHub repository [here](https://github.com/daveihart/python-aws-lambda-ec2_reporter).