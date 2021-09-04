---
title: 'Encrypting EBS volumes programmatically with python'
date: Mon, 26 Oct 2020 18:42:52 +0000
draft: true
tags: ['AWS', 'AWS', 'Blog', 'ebs', 'encryption', 'GitHub', 'GitHub', 'python', 'python']
author: "Dave"
toc: true
featured_image: "image/title/aws_ebs_encryption.png"
---

Encrypting attached AWS EBS volume involves a number of steps. This article will show you how to encrypt your volumes using python.

Let's set the scene, you have an environment hosting a number of AWS EC2 instances and now security have said, "Hey, these EBS volumes should be encrypted!" No argument from me. So how do we go about this programmatically.

{{< imgproc info Resize "100x" >}}

_**You can enable default volume encryption in the management console. Check this [link](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSEncryption.html#encryption-by-default) out on how to do it. It was not always there hence the need for this process.**_

A word on reuse of code
-----------------------

Just popping this in as if you looked at my article on [terminating EC2 instances by tag](/post/terraform-aws-and-wordpress-part-ii/), you will notice there is a fair bit of code reused here. A phrase I hear quite a lot over the years is 'let's not re-invent the wheel!' So whilst this is fantastic advice, it sometimes takes longer finding where the wheel was last left! In my own development work I found that sometimes I could not find a snippet of code I once used. Thankfully these days we have things like online git repositories.

Getting back on topic....

Method.
-------

Unfortunately (at the time of writing) there is no easy method of changing an EBS volume from being unencrypted to encrypted. The process is along these lines

1.  Shutdown the instance (if running)
2.  Create a snapshot (This will be unencrypted) of the EBS volume
3.  Copy the snapshot and enable encryption of the copy
4.  Create a new EBS volume from the encrypted snapshot
5.  Detach the unencrypted EBS volume and attach the encrypted volume
6.  Start the instance (if it was running)
7.  Optional: Delete the unencrypted EBS volumes

Hopefully you thought the same as me, you definitely do **not** want to be doing this manually!

Encryption Key
--------------

The example in this guide uses the AWS accounts default EBS encryption key. Should you for any reason want to share the encrypted volumes or snapshots with another account you cannot use the default key. It's the default key for the specific AWS account the key resides. You could create your own key and encrypt using that. The key you create can then be shared with other accounts.

The image below shows the default (AWS Managed) ebs encryption key.

{{< figure src="image-51.png" >}}

Process Flow
------------

This is another one of those developed in bash for a [GoCD](https://www.gocd.org/) pipeline scripts which I am again rewriting into Python.

We already have a feel for the steps involved listed in the method section above, let's take a look at the flowchart.

### Flowchart - Full

{{< figure src="ec2-terminate-flow.png" >}}

You will notice this is the same flow diagram used in the article [Terminate EC2 instance by tag](/post/encrypting-ebs-volumes-programmatically-python/) . The flow will be the same, the "Process Instances" procedure is where are the interesting stuff with be done. Let's dive into that process

### Flowchart - Process Instances

{{< figure src="ebs_encryption_detailed_flow.jpeg" >}}

As you can see, not a straightforward process.

Script
------

Similar to my previous scripts we break it down into functions. I will also include remarks to elaborate on some of the code. Not everyone will want or need the remarks so there will be a link at the end to the public github repository which is a lot cleaner.

### Imports

This first block of code is setting the scene by **_importing_** libraries or only parts of a library by using **_from_**.

```python
#!/usr/bin/env python3.8
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

### Variables

These are the variables we can use in any CD package

```python
search_tag = "some_key_name" # Tag to search for instances
search_value = "some_key_value"  #value in tag to search for instances 
snap_prefix = dtstr+"-post_encryption-snapshot" #snapshot description
arole="AddARoleHere" #role to assume across accounts
accounts = ['AWS Accounts Here', 'Another AWS Account here'] # list of accounts e.g ['0000000000000','1111111111111','2222222222222222','333333333333333333']
total_acc = len(accounts)
region = "eu-west-2"
verbose = True #False will suppress most of the output
```

Functions
---------

#### **Main**

This is the function which ties it all together. It handles the calls to the other functions.

```python
def main():
    global ec2
    global instances
    global az2
    global Iid
    global inst
    global FailedIid
    global SuccessIid
    processing_acc = 0
    FailedIid = []
    SuccessIid = []
    client = boto3.client("sts")
    account_id = client.get_caller_identity()["Account"]
    if verbose:
        print(f"script is executing in {account_id}")
    for acc in accounts:
        processing_acc += 1
        if verbose:
            print(f"Processing account : {processing_acc}")
        if acc != account_id:
            assume_roles(acc,accounts,arole)
            ec2 = boto3.client('ec2',aws_access_key_id=acc_key,aws_secret_access_key=sec_key,aws_session_token=sess_tok,region_name='eu-west-1')
        else:
            if verbose:
                print(f"Execution account, no assume required")
            ec2=boto3.client('ec2')
        instances = get_instances(processing_acc)
        for inst in instances:
            try:
                az=inst.get("Placement")
                az2=az.get("AvailabilityZone")
                Iid = []
                Iid.append(inst.get('InstanceId'))
                #The for and first if can be removed, used to retrieve the name tag for verbose output
                for tags in inst.get('Tags'):
                    if tags["Key"] == 'Name':
                        Iname = tags["Value"]
                        process_instance(Iname)
                print("-------------------------------------------------------------------------")
            except botocore.exceptions.ClientError as er:
                print("error on main")
                print(er.response['Error']['Message'])
                FailedIid.append(Iid[0])
                continue
    print(f"Errors encountered with these instances: {FailedIid}")
    print(f"Successfully processed these instances: {SuccessIid}")

if __name__ == "__main__":
    main()
```

### **assume\_roles**

This is exactly the same as my reference script. Roles are good. Use roles! Bit like cake, I like cake!

If the AWS account being processed is not the AWS account the script is being executed from, you will need to assume a role defined in the target account. The role will have to have sufficient access to perform the necessary EC2 operations. In the source AWS account the instance, person or service performing the execution will need access to use the AWS Security Token Service (STS). AWS STS will grant temporary short lived credentials to perform the necessary activities. This is the recommended approach and removed the need to have credentials stored in code.

```python
def assume_roles(acc,accounts,arole):
    global acc_key
    global sec_key
    global sess_tok
    global client
    sts_conn = boto3.client('sts')
    tmp_arn = f"{acc}:role/{arole}"
    response = sts_conn.assume_role(DurationSeconds=900,RoleArn=f"arn:aws:iam::{tmp_arn}",RoleSessionName='Test')
    acc_key = response['Credentials']['AccessKeyId']
    sec_key = response['Credentials']['SecretAccessKey']
    sess_tok = response['Credentials']['SessionToken']
```

### **get\_instances**

Another blatant re-use, handy this ;) Once you have worked out the AWS account security settings , you can start retrieving the EC2 instances for processing. The EC2 instances retrieved are filtered to only retrieve instances matching the **search_tag** and **search_value**. These are then added to a list for further processing.

```python
def get\instances(process_acc,filters=[{'Name': 'tag:'+search_tag, 'Values': [search_value]}]):
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

### process\_instances

This is the function which justified its own flowchart above. All of the processing of the EC2 instance is performed within this function and it makes all the calls to the functions which handle the snapshots, detaching, attaching etc.

```
def process\_instance(Iname):
    global volatt
    global volid
    global tags\_list
    global Istate
    global IstateCode
    processed\_status = "no"    #set a value to establish if the instance state has been retrieved
    Istate = inst.get('State')  
    IstateCode = Istate.get('Code')
    get\_status(Iid)
    if verbose:
        print(f"Instance Name : {Iname} ; Instance Id : {Iid\[0\]} ; Instance state : {IstateCode}")
        print(f"Checking volumes attached to {Iid\[0\]} for encryption settings")
    vols = ec2.describe\_volumes(
        Filters=\[
        {
            'Name': 'attachment.instance-id',
            'Values': \[
                str(Iid\[0\]),
           \],
        },
        \],
    )
    x = 0
    for dev in vols.get('Volumes'):    #Loop through each EBS volumes attached to the EC2 instance
        x = x +1   #used to increment and show the volume number being processed
        if verbose:
            print(f"processing volume : {x}")
        try:
            att = dev.get("Attachments")   #retrieve the attachment list in the volume json
            encstatus = dev.get('Encrypted')   #retrieve the encryption state
            volid = dev.get ('VolumeId')
            if verbose:
                print(f"volumeid :      {volid}")
                print(f"attachment :    {att}")
            volatt = att\[0\].get('Device')
            if encstatus == False:   #its not encrypted so lets sort that out, its why we are here :)
                if verbose:
                    print(f"Volume will need to be encrypted")
                if initial\_status == "running" and processed\_status == "no":   #if the instance is running, shut it down. You only want to do this once and not iterate through it with each volume
                    if verbose:
                        print(f"shutting down {Iid\[0\]}")
                    shutdown\_instance(Iid)  #call the shutdown instance function
                    processed\_status = "yes"   #set it as processed so we don't try shut it down again for the next volume
                tags\_list = dev.get('Tags', \[\])   #We need the tags to add them to the new encrypted volume
                moveon="no"
                while not Iid\[0\] in FailedIid and moveon == "no":   #step through this section until one of two things happen. The instance is added to the failure list or we set moveonto yes
                    detach\_old\_ebs()   #function to detach the old EBS volume
                    snapshot\_volumes()   #function to create the unencrypted snapshot
                    snapshot\_copy()   #function to create an encrypted copy of the unencrypted snapshot and delete the unencrypted snapshot
                    create\_ebs()   #function to create an encrypted EBS volume from the encrypted snapshot
                    attach\_new\_ebs()   #attach the new encrypted EBS volume
                    set\_delete\_terminate()   #for good messure set the deleteonterminate flag to true
                    delete\_ebs()   #delete the old unencrypted ebs
                    moveon="yes"   #flag to exit the while loop
            else:
                if verbose:
                    print(f"{volid} is already encrypted")
        except botocore.exceptions.ClientError as er:   #capture and output any exceptions
            print("error on check\_volumes")
            print(er.response\['Error'\]\['Message'\])
            FailedIid.append(Iid\[0\])
    if initial\_status == "running":
        if verbose:
            print(f"starting instance {Iid\[0\]}")
        start\_instance(Iid)   #if the instance was running when we started processing, start it back up
    else:   #the EC2 instance was stopped when we started. We got this far so lets mark it as a success.
        if not Iid\[0\] in FailedIid:
            SuccessIid.append(Iid\[0\])
```

The next few functions are all called by the preceding function so I will try keep them in a logical order starting of with retrieving the EC2 instances run state.

### get\_status

As this script will be stopping and starting instances you want to know what the starting state was. Why? you might ask. well let's say you were using the cloud in the correct manner, it's possible that the instances are already in a stopped state for cost saving purposes. You want to be able to leave them like that when you are finished. This function will work out the state based on the meta state codes. The variable names _initial\_status_ will be returned to the calling function.

```
def get\_status(Iid):
    global initial\_status
    if IstateCode == 16 or IstateCode ==32 or IstateCode == 64 or IstateCode == 80:
        if IstateCode == 16:
            initial\_status = "running"
        elif IstateCode == 32 or IstateCode == 64:
            initial\_status = "shutting\_down"
        elif IstateCode == 80:
            initial\_status = "stopped"
    else:
            print(f"Warning : Instance {Iid\[0\]} is not running, stopping or stopped. Please perform a manual check")
            initial\_status = "not\_sure"
            FailedIid.append(Iid\[0\])
    return initial\_status #Capture and return the status to the calling function
```

### **shutdown\_instance**

Shutdown the EC2 instance being processed.

![](https://davehart.co.uk/wp-content/uploads/2020/09/information-1015297_1280-1024x1024.jpg)

#### error handling

You might notice a number of try and except statements in the functions throughout this example. Try something, if it fails, capture and output the error. All part of improving my code and learning something new. Good link [here](https://docs.python.org/3/tutorial/errors.html) on this very subject.

```
def shutdown\_instance(Iid):
    try:
        ec2.stop\_instances(InstanceIds=Iid)
        shutdown\_instance\_wait(Iid,initial\_status)
    except botocore.exceptions.ClientError as er:
        print(er.message)
        print("error on shutdown\_instance")
        FailedIid.append(Iid\[0\])

```

### **shutdown\_instance\_wait**

call the waiter to ensure you do not try to detach or attach disks whilst the EC2 instance is running. That will not work!

e.g. the _class _EC2.Waiter.InstanceTerminated polls [EC2.Client.describe\_instances()](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2.html#EC2.Client.describe_instances) every 15 seconds until a successful state is reached. An error is returned after 40 failed checks (10 minutes if you were working it out).

```
def shutdown\_instance\_wait(Iid):
    shutdown\_instance\_waiter = ec2.get\_waiter('instance\_stopped')
    try:
        shutdown\_instance\_waiter.wait(InstanceIds=Iid)
        if verbose:
            print(f"Instance {Iid\[0\]} has shutdown successfully")
    except botocore.exceptions.WaiterError as er:
        if "Max attempts exceeded" in er.message:
            print("error on shutdown\_instance\_wait")
            print(f"Instance {Iid\[0\]} did not shutdown in 600 seconds")
            FailedIid.append(Iid\[0\])
        else:
            print("error on shutdown\_instance\_wait\_else")
            print(er.message)
            FailedIid.append(Iid\[0\])
    return initial\_status
```

### detach\_old\_ebs

Now that the instance is safely stopped we can detach the EBS volume being processed.

```
def detach\_old\_ebs():
    try:
        global ebs\_wait
        detach\_ebs = ec2.detach\_volume(
        Device=volatt,
        InstanceId=Iid\[0\],
        VolumeId=volid,
        )
        if verbose:
            print(f"Waiting for volume {volid} to be detached")
        ebs\_wait = volid
        create\_ebs\_wait()
    except botocore.exceptions.ClientError as er:
        print("error on detach\_old\_ebs")
        print(er.response\['Error'\]\['Message'\])
        FailedIid.append(Iid\[0\])
```

### create\_ebs\_wait

We need to ensure the volume is detached before we proceed with either trying to attach another volume to its device attachment or before trying to terminate it. This waiter turned into a bit of a pain. The proceeding steps would not always work as they were seeing the original EBS as still attached. In the end the introduction of a 5 second timer resolved the issue which implies the waiter is not that reliable for the "volume\_available" wait! No tip for him! (Yes, I was dying to say that :) ). We will use this waiter again when we attach the new EBS volume

```
def create\_ebs\_wait():
    global ebs\_check
    ebs\_check = \[\]
    ebs\_check.append(ebs\_wait)
    try:
        create\_ebs\_available\_waiter = ec2.get\_waiter('volume\_available')
        create\_ebs\_available\_waiter.wait(VolumeIds=ebs\_check)  
        time.sleep(5)   #Not impressed that I needed to use this. googling proved I was not alone in this
    except botocore.exceptions.WaiterError as er:
        if "Max attempts exceeded" in er.message:
            print(f"Volume {enc\_ebs\_id} was not available in the max wait time")
        else:
            print("error on create\_ebs\_wait")
            print(er.message)
            FailedIid.append(Iid\[0\])    
```

### snapshot\_volumes

Create an unencrypted snapshot of the now detached unencrypted EBS volume. Add the tags we retrieved from the EBS volume to the snapshot.

```
def snapshot\_volumes():
    global snap\_shot
    global unenc\_snapshot
    try:
        snap\_shot = ec2.create\_snapshot(
            VolumeId=volid,
            Description=snap\_prefix,
            TagSpecifications=\[
                {
                'ResourceType' : 'snapshot',
                'Tags': tags\_list,      
                }
            \],
        )
        create\_snapshots\_wait(snap\_shot)
        unenc\_snapshot = snap\_shot.get('SnapshotId')
        if verbose:
            print(f"Unencrypted snapshot : {unenc\_snapshot}")
    except botocore.exceptions.ClientError as er:
        print("error on snapshot\_volumes")
        print(er.response\['Error'\]\['Message'\])
        FailedIid.append(Iid\[0\])

```

### create\_snapshots\_wait

This function will be called by the snapshot\_volumes function and also the snapshot\_copy function. Both of these rely upon the same waiter, snapshot\_completed

```
def create\_snapshots\_wait(snap\_shot):
    global snap\_check
    snap\_check = \[\]
    snap\_check.append(snap\_shot.get('SnapshotId'))
    try:
        create\_snapshot\_waiter = ec2.get\_waiter('snapshot\_completed')
        if verbose:
            print(f"Waiting for {snap\_check\[0\]}")
        create\_snapshot\_waiter.wait(SnapshotIds=snap\_check)  
    except botocore.exceptions.WaiterError as er:
        if "Max attempts exceeded" in er.message:
            print("error on create\_snapshots\_wait")
            print(f"Instance {Iid\[0\]} did not shutdown in 600 seconds")
            FailedIid.append(Iid\[0\])
        else:
            print("error on create\_snapshots\_wait\_else")
            print(er.message)
            FailedIid.append(Iid\[0\])
```

### snapshot\_copy

Now we make use of the unencrypted snapshot and create a copy turning on encryption. Again we make use of the original EBS volume tags. We call the preceding wait function again.

```
def snapshot\_copy():
    global enc\_snapshot
    try:
        snap\_shot = ec2.copy\_snapshot(
        Description="encrypted-"+snap\_prefix,
        TagSpecifications=\[
                {
                'ResourceType' : 'snapshot',
                'Tags': tags\_list,      
                }
            \],  
        Encrypted=True,
        SourceRegion=region,
        SourceSnapshotId=snap\_check\[0\],
        )
        enc\_snapshot = snap\_shot.get('SnapshotId')
        create\_snapshots\_wait(snap\_shot)
        if verbose:
            print(f"encrypted snapshot : {enc\_snapshot}")
    except botocore.exceptions.ClientError as er:
        print("error on snapshot\_copy")
        print(er.response\['Error'\]\['Message'\])
        FailedIid.append(Iid\[0\])
    ec2.delete\_snapshot(SnapshotId=unenc\_snapshot,
    )
```

### create\_ebs

We now make use of the encrypted snapshot to make an encrypted EBS volume

```
def create\_ebs():
    global enc\_ebs\_id
    try:
        enc\_ebs = ec2.create\_volume(
        AvailabilityZone=az2,
        Encrypted=True,
        SnapshotId=enc\_snapshot,
        TagSpecifications=\[
                {
                'ResourceType' : 'volume',
                'Tags': tags\_list,      
                }
            \],
        )
        enc\_ebs\_id = enc\_ebs.get('VolumeId')
        ebs\_wait = enc\_ebs\_id
        create\_ebs\_wait()
    except botocore.exceptions.ClientError as er:
        print("error on create\_ebs")
        print(er.message)
        FailedIid.append(Iid\[0\])
```

We call the previously describe waiter we used for the EBS volume detach function

### attach\_new\_ebs

We are as sure as we can be (without adding a further check) that the attachment is now free so we can attach the new EBS volume

```
def attach\_new\_ebs():
    try:
        if verbose:
            print(f"Attaching volume {enc\_ebs\_id} to {volatt}")
        attach\_ebs = ec2.attach\_volume(
        Device=volatt,
        InstanceId=Iid\[0\],
        VolumeId=enc\_ebs\_id,
        )
    except botocore.exceptions.ClientError as er:
        print("error on attach\_new\_ebs")
        print(er.response\['Error'\]\['Message'\])
        FailedIid.append(Iid\[0\])
```

### set\_delete\_terminate

Ensures that the EBS volumes have the value deleteonetermination set to true. Without this you can end up with a lot of orphaned EBS volumes. It's not required for volume encryption just some housekeeping.

```
def set\_delete\_terminate():
    if verbose:
        print(f"deleteontermination check : {enc\_ebs\_id}")
    delonterm = ec2.modify\_instance\_attribute(
    Attribute='blockDeviceMapping',
    BlockDeviceMappings=\[
    {
    'DeviceName': volatt,
    'Ebs': {
    'DeleteOnTermination': True,
    }}\],
    InstanceId=Iid\[0\])
```

### delete\_ebs

This one depends on how much confidence you have. It will terminate the original unencrypted EBS volume...gulp (You should already have a backup / snapshot lifecycle in place to give you that warm feeling)

```
def delete\_ebs():
    try:
        delete\_ebs = ec2.delete\_volume(
            VolumeId=volid
        )
    except botocore.exceptions.ClientError as er:
        print("error on delete\_ebs")
        print(er.response\['Error'\]\['Message'\])
        FailedIid.append(Iid\[0\])
```

### start\_instance

And finally you want to start the instance back up if this was the original state.

```
def start\_instance(Iid):
    try:
        ec2.start\_instances(InstanceIds=Iid)
        SuccessIid.append(Iid\[0\])
    except botocore.exceptions.ClientError as er:
        print("error on start\_instance")
        print(er.response\['Error'\]\['Message'\])
        FailedIid.append(Iid\[0\])
```

Conclusion
----------

There we go, a python script which retrieves AWS EC2 instances across multiple AWS Accounts assuming roles where required. The instances are checked for unencrypted EBS volumes, if found they are processed to provide the end result of encrypted EBS volumes.

The script will output a list of successful and failed instance id's for checking.

As usual, [here](https://github.com/daveihart/python-aws-ebs_encryption_public) is a link to my GitHub repo containing the source code. Feel free to ask any question or suggest any improvements.

Till next time.

Disclaimer: This script will terminate EBS volumes. Use at your own risk!