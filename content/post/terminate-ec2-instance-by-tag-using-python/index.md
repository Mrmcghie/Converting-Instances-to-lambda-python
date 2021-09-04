---
title: 'Terminate EC2 instance by tag using Python'
date: Fri, 25 Sep 2020 17:35:22 +0000
draft: false
tags: ['AWS', 'AWS', 'Blog', 'boto3', 'deleteontermination', 'ec2', 'GitHub', 'GitHub', 'python', 'python', 'snapshot', 'tags', 'waiters']
author: "Dave"
toc: true
featured_image: "image/title/python-aws-terminate-ec2.png"
---

I originally developed this script in bash for a [GoCD](https://www.gocd.org/) pipeline. The intention of the script is to decommission instances in batch whilst generating a final snapshot. For those of you that wonder why is Terraform not being used? (You know who you are) Good question. Not everyone is using Terraform yet and refactoring/importing deployment methods will incur costs which some clients are not prepared to pay.

Why change it?
--------------

The move to Python was a logical one for me, it removes any OS dependency and I also see this as good excuse to practice my Python. If you don't get to use a skill a lot you tend to lose it. The previous bash script used an input file and those nasty access keys (Not exactly secure). This one will use tags and roles. There were also some enhancements AWS had introduced since the original script was written which would optimise the code, waiters for example.

Considerations.
---------------

I considered the following activities as part of the script execution.

*   The tags need to be clearly defined to only target the instances required.
*   I should first snapshot the instance.
*   Actually I should check if still running and if so stop it. ([Best practice](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-creating-snapshot.html) for root volumes)
*   Then take a snapshot
*   Wait! I should confirm it has shutdown before taking the snapshot (enter the waiter)
*   Now let's take a snapshot
*   Then I can terminate the instance
*   hmm, I should wait for the snapshots to complete first (waiter is back)
*   Now I can terminate the instance
*   What if the EBS volume was created so that it does not terminate on deletion of the instance (**_deleteontermination=False_**). Best add a check for that and change to True if found.

Got there in the end but that was how I initially planned it on the whiteboard.

Flowchart
---------

When writing any code I find a diagram helps me focus on the process flow.

{{< figure src="ec2-terminate-flow.png" >}}

Script
------

The script is broken down into multiple functions. It makes a lot of sense this way as you only need to write a block of code once and can call it many times. Additionally it's easier to pinpoint issues with your code. GitHub Link at the end for the full code.

### Imports

This first block of code is setting the scene by **_importing_** libraries or only parts of a library by using **_from_**

```
#!/usr/bin/env python3.6
import botocore
import boto3
from boto3.session import Session
from botocore.exceptions import ClientError
import time
import os
from datetime import date
dt=date.today()
dtstr=dt.strftime("%Y-%m-%d")
```

{{< imgproc info Resize "100x" >}}

Boto3 is the Amazon Web Services (AWS) Software Development Kit (SDK) for Python, which allows Python developers to write software that makes use of services like Amazon S3 and Amazon EC2.

*source https://github.com/boto/boto3*

### Variables

Now we set some parameters to use throughout the script. If you were executing this from a continuous deployment package such as Jenkins, then most of these could be defined as Environment Variables.

```
dry\_run=False
search\_tag = "Name" # Tag to search for instances
search\_value = "dtest\*"  #value in tag to search for instances 
snap\_prefix = dtstr+"-pre-termination-snapshot" #snapshot description
arole="AddARoleHere" #role to assume across accounts
accounts = \['0000000000000','1111111111111'\] # list of accounts e.g \['0000000000000','1111111111111','2222222222222222','333333333333333'\]
#End of variables
total\_acc = len(accounts)
```

### Functions

There are a number of functions to walk through.

#### **assume\_roles**

If the AWS account being processed is not the AWS account the script is being executed from, you will need to assume a role defined in the target account. The role will have to have sufficient access to perform the necessary EC2 operations. In the source AWS account the instance, person or service performing the execution will need access to use the AWS Security Token Service (STS). AWS STS will grant temporary short lived credentials to perform the necessary activities. This is the recommended approach and removed the need to have credentials stored in code.

```
def assume\_roles(acc,accounts,arole):
    global acc\_key
    global sec\_key
    global sess\_tok
    global client
    sts\_conn = boto3.client('sts')
    tmp\_arn = f"{acc}:role/{arole}"
    response = sts\_conn.assume\_role(DurationSeconds=900,RoleArn=f"arn:aws:iam::{tmp\_arn}",RoleSessionName='Test')
    acc\_key = response\['Credentials'\]\['AccessKeyId'\]
    sec\_key = response\['Credentials'\]\['SecretAccessKey'\]
    sess\_tok = response\['Credentials'\]\['SessionToken'\]
```

#### **get\_instances**

Once you have worked out the AWS account security settings , you can start retrieving the EC2 instances for processing. The EC2 instances retrieved are filtered to only retrieve instances matching the search\_tag and search\_value. These are then added to a list for further processing.

```
def get\_instances(process\_acc,filters=\[{'Name': 'tag:'+search\_tag, 'Values': \[search\_value\]}\]):
    reservations = {}
    try:
        reservations = ec2.describe\_instances(
            Filters=filters
        )
    except botocore.exceptions.ClientError as e:
        print(e.response\['Error'\]\['Message'\])
    instances = \[\]
    for reservation in reservations.get('Reservations', \[\]):
        for instance in reservation.get('Instances', \[\]):
            instances.append(instance)
    return instances 
```
{{< imgproc info Resize "100x" >}}

**_A waiter will automatically poll AWS resources for pre-defined status changes. These will perform a check every x seconds for x number of times. x varies depending on the [waiter](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2.html#waiters) being used._**

#### **shutdown\_instance & shutdown\_instance\_wait**

Easy one, shutdown the EC2 instance and then call the waiter.

e.g. the _class _EC2.Waiter.InstanceTerminated polls [EC2.Client.describe\_instances()](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2.html#EC2.Client.describe_instances) every 15 seconds until a successful state is reached. An error is returned after 40 failed checks (10 minutes if you were working it out).

```
def shutdown\_instance(Iid):
    ec2.stop\_instances(InstanceIds=Iid)
    shutdown\_instance\_wait(Iid)

def shutdown\_instance\_wait(Iid):
    shutdown\_instance\_waiter = ec2.get\_waiter('instance\_stopped')
    try:
        shutdown\_instance\_waiter.wait(InstanceIds=Iid)
    except botocore.exceptions.WaiterError as er:
        if "Max attempts exceeded" in er.message:
            print(f"Instance {Iid\[0\]} did not shutdown in 600 seconds")
        else:
            print(er.message)

```

#### **snapshot\_volumes & create\_snapshots\_wait**

This function creates a snapshot of all EBS volumes attached to the specific EC2 instance (You used to have to enumerate each volumes programmatically. The addition of the **s** at the end of **create\_snapshots** is welcome). This will also copy any tags defined on the source EBS volumes to the snapshot. Nice touch. The waiter is just waiting for each snapshot to complete. You will see that I had to add all the snapshot ID's to a list. This is then used by the waiter.

```
def snapshot\_volumes(Iid,inst):
    snap\_check=\[\]
    snap\_shot = ec2.create\_snapshots(
        Description=snap\_prefix,
        InstanceSpecification={
            'InstanceId': Iid\[0\],
            'ExcludeBootVolume': False
        },
        DryRun=dry\_run,
        CopyTagsFromSource='volume'
    )
    #build a list of snapshotids so we can check if they are complete - waiter
    for snapid in snap\_shot.get('Snapshots'):
        snap\_check.append(snapid.get('SnapshotId'))
    create\_snapshots\_wait(snap\_check)

def create\_snapshots\_wait(snap\_check):
    try:
        create\_snapshot\_waiter = ec2.get\_waiter('snapshot\_completed')
        create\_snapshot\_waiter.wait(SnapshotIds=snap\_check)  
    except botocore.exceptions.WaiterError as er:
        if "Max attempts exceeded" in er.message:
            print(f"Instance {Iid\[0\]} did not shutdown in 600 seconds")
        else:
            print(er.message)
```

#### **check\_volumes**

As this exercise is all about cleaning up EC2 instances it would be remiss to leave a load of unattached (orphaned) EBS volumes behind. By default a root volume will be automatically deleted on EC2 instance termination but the same cannot be said for EBS volumes added afterwards. To remove these we need to ensure the EBS volumes **_"deleteontermination"_** attribute is set as True.

```
def check\_volumes(Iid,inst):
    terminate\_check=\[\]
    for bdevice in inst.get('BlockDeviceMappings'):
        vol = bdevice.get('Ebs') #type
        volatt = bdevice.get('DeviceName') #attachment, always useful
        delter = vol.get('DeleteOnTermination')
        if delter == False:
            delonterm = ec2.modify\_instance\_attribute(
                Attribute='blockDeviceMapping',
                BlockDeviceMappings=\[
                    {
                        'DeviceName': volatt,
                        'Ebs': {
                            'DeleteOnTermination': True,
                            }}\],
                            InstanceId=Iid\[0\])
        else:
            print(f"deleteontermination check : EBS will be deleted, no change required")
```

#### **terminate\_instances**

As long as all the other functions have completed we will now be terminating the EC2 instance.

```
def terminate\_instances(Iid,FailedIid,SuccessIid):
    try:
        terminate\_instance = ec2.terminate\_instances(
            InstanceIds=Iid,
            DryRun=dry\_run
        )
        SuccessIid.append(Iid\[0\])
    except ClientError as er:
        FailedIid.append(Iid\[0\])
        print(f"an error occured terminating {Iid\[0\]} - {err.message}")
```

#### **Main**

This is the function which ties it all together. It handles the calls to the other functions at the right part of the process.

```
def main():
    global ec2
    global instances
    processing\_acc = 0
    FailedIid = \[\]
    SuccessIid = \[\]
    #retrieve account context the script is executing so we know whether to assume a role or not
    client = boto3.client("sts")
    account\_id = client.get\_caller\_identity()\["Account"\]
    for acc in accounts:
        processing\_acc += 1
        if acc != account\_id:
            assume\_roles(acc,accounts,arole)
            ec2 = boto3.client('ec2',aws\_access\_key\_id=acc\_key,aws\_secret\_access\_key=sec\_key,aws\_session\_token=sess\_tok,region\_name='eu-west-1')
        else:
            print(f"Execution account, no assume required")
            ec2=boto3.client('ec2')
        instances = get\_instances(processing\_acc)
        for inst in instances:
            Iid = \[\]
            Iid.append(inst.get('InstanceId'))
            Istate = inst.get('State')
            IstateCode = Istate.get('Code')
            #The for and first if can be removed, used to retrieve the name tag for verbose output
            for tags in inst.get('Tags'):
                if tags\["Key"\] == 'Name':
                    Iname = tags\["Value"\]
                    print(f"Instance Name : {Iname} ; Instance Id : {Iid\[0\]} ; Instance state : {IstateCode}")
                    #EC2 Instance States = 0 : pending ; 16 : running ; 32 : shutting-down ; 48 : terminated ; 64 : stopping ; 80 : stopped
                    if IstateCode == 16 or IstateCode ==32 or IstateCode == 64 or IstateCode == 80:
                        if IstateCode == 16:
                            print("Instance is running") # proceed to shut it down
                            shutdown\_instance(Iid)
                        elif IstateCode == 32 or IstateCode == 64: 
                            print("Instance is already shutting down, calling waiter")
                            shutdown\_instance\_wait(Iid)
                        elif IstateCode == 80:
                            print(f"Instance is stopped")
                        check\_volumes(Iid,inst)
                        snapshot\_volumes(Iid,inst)
                        terminate\_instances(Iid,FailedIid,SuccessIid)
                    else:
                        print(f"Warning : Instance {Iid\[0\]} is not running, stopping or stopped. Please perform a manual check")
                        FailedIid.append(Iid\[0\])
    print(f"Terminated : {SuccessIid}")
    print(f"Failed to Terminate : {FailedIid}")
            
if \_\_name\_\_ == "\_\_main\_\_":
    main()
```

Testing
-------

Testing was performed during and after the code generation. For the final bout of testing I created a few EC2 instances across multiple AWS accounts in varying states. These had a mixture of 1-4 EBS volumes attached.

The instances all had the "Name" tag "dtestx" where x was a numeric value.

Name | Account | Status | EBS No. |
---- | ------- | ------ | ------- |
dtest1 | 1 | Running | 2 |
dtest2 | 1 | Stopped | 1 |
dtest3 | 2 | Running | 4 |
dtest4 | 2 | Running | 2 |

> Using the "_Name_" tag was only for example sake. I recommend a dedicated unique tag Key and tag Value

I have extracted the output below for an example. It may seem a bit verbose for some, I personally like to get as much info as I can from my scripts. You will see that the last two lines from the output identify the EC2 instances either successful or unsuccessfully processed.

```text
prompt:~/scripts/python-aws-ec2\_cleanup$ source env/bin/activate
(env) prompt:~/scripts/python-aws-ec2\_cleanup$ python3.6 ec2cleanup1.0.py
2 AWS account(s) detected
Processing account : 1
Execution account, no assume required
Instance Name : dtest2 ; Instance Id : i-0550d3810abdcced0 ; Instance state : 80
Instance is stopped
Checking volumes attached to ['i-0550d3810abdcced0'] for Termination settings
deleteontermination check : EBS will be deleted, no change required
Processing volumes attached to i-0550d3810abdcced0 for snapshot
Waiting for ['snap-000bedcc242048999']
Proceding with termination of i-0550d3810abdcced0
Instance Name : dtest1 ; Instance Id : i-0624c25c58820f3bf ; Instance state : 16
Instance is running, shutting down prior to creating snapshot(s) of attached volume(s)
Instance i-0624c25c58820f3bf has shutdown successfully
Checking volumes attached to ['i-0624c25c58820f3bf'] for Termination settings
deleteontermination check : EBS is not set to delete on termination, lets fix that
deleteontermination check : EBS is not set to delete on termination, lets fix that
Processing volumes attached to i-0624c25c58820f3bf for snapshot
Waiting for ['snap-0f31c5a78b0d46473', 'snap-073bc00fc605141c0']
Proceding with termination of i-0624c25c58820f3bf
Processing account : 2
Assuming role
Instance Name : dtest3 ; Instance Id : i-013db3cc3fe34604b ; Instance state : 16
Instance is running, shutting down prior to creating snapshot(s) of attached volume(s)
Instance i-013db3cc3fe34604b has shutdown successfully
Checking volumes attached to ['i-013db3cc3fe34604b'] for Termination settings
deleteontermination check : EBS will be deleted, no change required
deleteontermination check : EBS is not set to delete on termination, lets fix that
deleteontermination check : EBS is not set to delete on termination, lets fix that
deleteontermination check : EBS is not set to delete on termination, lets fix that
Processing volumes attached to i-013db3cc3fe34604b for snapshot
Waiting for ['snap-009df192e5f81bc55', 'snap-0ce2c45da6667dd6f', 'snap-01e2cded42c045d44', 'snap-085152d1458e55fa1']
Proceding with termination of i-013db3cc3fe34604b
Instance Name : dtest4 ; Instance Id : i-042b3f3c62782d80a ; Instance state : 16
Instance is running, shutting down prior to creating snapshot(s) of attached volume(s)
Instance i-042b3f3c62782d80a has shutdown successfully
Checking volumes attached to ['i-042b3f3c62782d80a'] for Termination settings
deleteontermination check : EBS will be deleted, no change required
deleteontermination check : EBS is not set to delete on termination, lets fix that
Processing volumes attached to i-042b3f3c62782d80a for snapshot
Waiting for ['snap-00e3a33ef15cb4537', 'snap-0886e3351306f75c7']
Proceding with termination of i-042b3f3c62782d80a
Terminated : ['i-0550d3810abdcced0', 'i-0624c25c58820f3bf', 'i-013db3cc3fe34604b', 'i-042b3f3c62782d80a']
Failed to Terminate : []
```

Conclusion
----------

So to wrap up. We have executed a python script which retrieves AWS EC2 instances across multiple AWS Accounts assuming roles where required. The instances are shutdown, snapshots taken and then terminated. EBS volume attributes are updated to ensure they are deleted as well.

I plan on documented my Terraform Jenkins deployment at some point, this may be a good execution script for testing.

[Here](https://github.com/daveihart/python-aws-terminate-ec2) is a link to my GitHub repo containing the source code. Have a play around (carefully). Feel free to throw any question across.

Till next time.

**Disclaimer: This script is designed to terminate ec2 instances. Use at your own risk!**