---
title: 'Encrypting EBS volumes programmatically with python'
date: Mon, 26 Oct 2020 18:42:52 +0000
draft: false
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

### **assume_roles**

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

### **get_instances**

Another blatant re-use, handy this ;) Once you have worked out the AWS account security settings , you can start retrieving the EC2 instances for processing. The EC2 instances retrieved are filtered to only retrieve instances matching the **search_tag** and **search_value**. These are then added to a list for further processing.

```python
def get_instances(process_acc,filters=[{'Name': 'tag:'+search_tag, 'Values': [search_value]}]):
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

### process_instances

This is the function which justified its own flowchart above. All of the processing of the EC2 instance is performed within this function and it makes all the calls to the functions which handle the snapshots, detaching, attaching etc.

```python
def process_instance(Iname):
    global volatt
    global volid
    global tags_list
    global Istate
    global IstateCode
    processed_status = "no"    #set a value to establish if the instance state has been retrieved
    Istate = inst.get('State')  
    IstateCode = Istate.get('Code')
    get_status(Iid)
    if verbose:
        print(f"Instance Name : {Iname} ; Instance Id : {Iid[0]} ; Instance state : {IstateCode}")
        print(f"Checking volumes attached to {Iid[0]} for encryption settings")
    vols = ec2.describe_volumes(
        Filters=[
        {
            'Name': 'attachment.instance-id',
            'Values': [
                str(Iid[0]),
           ],
        },
        ],
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
            volatt = att[0].get('Device')
            if encstatus == False:   #its not encrypted so lets sort that out, its why we are here :)
                if verbose:
                    print(f"Volume will need to be encrypted")
                if initial_status == "running" and processed_status == "no":   #if the instance is running, shut it down. You only want to do this once and not iterate through it with each volume
                    if verbose:
                        print(f"shutting down {Iid[0]}")
                    shutdown_instance(Iid)  #call the shutdown instance function
                    processed_status = "yes"   #set it as processed so we don't try shut it down again for the next volume
                tags_list = dev.get('Tags', [])   #We need the tags to add them to the new encrypted volume
                moveon="no"
                while not Iid[0] in FailedIid and moveon == "no":   #step through this section until one of two things happen. The instance is added to the failure list or we set moveonto yes
                    detach_old_ebs()   #function to detach the old EBS volume
                    snapshot_volumes()   #function to create the unencrypted snapshot
                    snapshot_copy()   #function to create an encrypted copy of the unencrypted snapshot and delete the unencrypted snapshot
                    create_ebs()   #function to create an encrypted EBS volume from the encrypted snapshot
                    attach_new_ebs()   #attach the new encrypted EBS volume
                    set_delete_terminate()   #for good messure set the deleteonterminate flag to true
                    delete_ebs()   #delete the old unencrypted ebs
                    moveon="yes"   #flag to exit the while loop
            else:
                if verbose:
                    print(f"{volid} is already encrypted")
        except botocore.exceptions.ClientError as er:   #capture and output any exceptions
            print("error on check_volumes")
            print(er.response['Error']['Message'])
            FailedIid.append(Iid[0])
    if initial_status == "running":
        if verbose:
            print(f"starting instance {Iid[0]}")
        start_instance(Iid)   #if the instance was running when we started processing, start it back up
    else:   #the EC2 instance was stopped when we started. We got this far so lets mark it as a success.
        if not Iid[0] in FailedIid:
            SuccessIid.append(Iid[0])
```

The next few functions are all called by the preceding function so I will try keep them in a logical order starting of with retrieving the EC2 instances run state.

### get_status

As this script will be stopping and starting instances you want to know what the starting state was. Why? you might ask. well let's say you were using the cloud in the correct manner, it's possible that the instances are already in a stopped state for cost saving purposes. You want to be able to leave them like that when you are finished. This function will work out the state based on the meta state codes. The variable names _initial_status_ will be returned to the calling function.

```python
def get_status(Iid):
    global initial_status
    if IstateCode == 16 or IstateCode ==32 or IstateCode == 64 or IstateCode == 80:
        if IstateCode == 16:
            initial_status = "running"
        elif IstateCode == 32 or IstateCode == 64:
            initial_status = "shutting_down"
        elif IstateCode == 80:
            initial_status = "stopped"
    else:
            print(f"Warning : Instance {Iid[0]} is not running, stopping or stopped. Please perform a manual check")
            initial_status = "not_sure"
            FailedIid.append(Iid[0])
    return initial_status #Capture and return the status to the calling function
```

### **shutdown_instance**

Shutdown the EC2 instance being processed.

{{< imgproc info Resize "100x" >}}

You might notice a number of try and except statements in the functions throughout this example. Try something, if it fails, capture and output the error. All part of improving my code and learning something new. Good link [here](https://docs.python.org/3/tutorial/errors.html) on this very subject.

```python
def shutdown_instance(Iid):
    try:
        ec2.stop_instances(InstanceIds=Iid)
        shutdown_instance_wait(Iid,initial_status)
    except botocore.exceptions.ClientError as er:
        print(er.message)
        print("error on shutdown_instance")
        FailedIid.append(Iid[0])
```

### **shutdown_instance_wait**

call the waiter to ensure you do not try to detach or attach disks whilst the EC2 instance is running. That will not work!

e.g. the _class _EC2.Waiter.InstanceTerminated polls [EC2.Client.describe_instances()](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2.html#EC2.Client.describe_instances) every 15 seconds until a successful state is reached. An error is returned after 40 failed checks (10 minutes if you were working it out).

```python
def shutdown_instance_wait(Iid):
    shutdown_instance_waiter = ec2.get_waiter('instance_stopped')
    try:
        shutdown_instance_waiter.wait(InstanceIds=Iid)
        if verbose:
            print(f"Instance {Iid[0]} has shutdown successfully")
    except botocore.exceptions.WaiterError as er:
        if "Max attempts exceeded" in er.message:
            print("error on shutdown_instance_wait")
            print(f"Instance {Iid[0]} did not shutdown in 600 seconds")
            FailedIid.append(Iid[0])
        else:
            print("error on shutdown_instance_wait_else")
            print(er.message)
            FailedIid.append(Iid[0])
    return initial_status
```

### detach_old_ebs

Now that the instance is safely stopped we can detach the EBS volume being processed.

```python
def detach_old_ebs():
    try:
        global ebs_wait
        detach_ebs = ec2.detach_volume(
        Device=volatt,
        InstanceId=Iid[0],
        VolumeId=volid,
        )
        if verbose:
            print(f"Waiting for volume {volid} to be detached")
        ebs_wait = volid
        create_ebs_wait()
    except botocore.exceptions.ClientError as er:
        print("error on detach_old_ebs")
        print(er.response['Error']['Message'])
        FailedIid.append(Iid[0])
```

### create_ebs_wait

We need to ensure the volume is detached before we proceed with either trying to attach another volume to its device attachment or before trying to terminate it. This waiter turned into a bit of a pain. The proceeding steps would not always work as they were seeing the original EBS as still attached. In the end the introduction of a 5 second timer resolved the issue which implies the waiter is not that reliable for the "volume_available" wait! No tip for him! (Yes, I was dying to say that :) ). We will use this waiter again when we attach the new EBS volume

```python
def create_ebs_wait():
    global ebs_check
    ebs_check = []
    ebs_check.append(ebs_wait)
    try:
        create_ebs_available_waiter = ec2.get_waiter('volume_available')
        create_ebs_available_waiter.wait(VolumeIds=ebs_check)  
        time.sleep(5)   #Not impressed that I needed to use this. googling proved I was not alone in this
    except botocore.exceptions.WaiterError as er:
        if "Max attempts exceeded" in er.message:
            print(f"Volume {enc_ebs_id} was not available in the max wait time")
        else:
            print("error on create_ebs_wait")
            print(er.message)
            FailedIid.append(Iid[0])    
```

### snapshot_volumes

Create an unencrypted snapshot of the now detached unencrypted EBS volume. Add the tags we retrieved from the EBS volume to the snapshot.

```python
def snapshot_volumes():
    global snap_shot
    global unenc_snapshot
    try:
        snap_shot = ec2.create_snapshot(
            VolumeId=volid,
            Description=snap_prefix,
            TagSpecifications=[
                {
                'ResourceType' : 'snapshot',
                'Tags': tags_list,      
                }
            ],
        )
        create_snapshots_wait(snap_shot)
        unenc_snapshot = snap_shot.get('SnapshotId')
        if verbose:
            print(f"Unencrypted snapshot : {unenc_snapshot}")
    except botocore.exceptions.ClientError as er:
        print("error on snapshot_volumes")
        print(er.response['Error']['Message'])
        FailedIid.append(Iid[0])
```

### create_snapshots_wait

This function will be called by the snapshot_volumes function and also the snapshot_copy function. Both of these rely upon the same waiter, snapshot_completed

```python
def create_snapshots_wait(snap_shot):
    global snap_check
    snap_check = []
    snap_check.append(snap_shot.get('SnapshotId'))
    try:
        create_snapshot_waiter = ec2.get_waiter('snapshot_completed')
        if verbose:
            print(f"Waiting for {snap_check[0]}")
        create_snapshot_waiter.wait(SnapshotIds=snap_check)  
    except botocore.exceptions.WaiterError as er:
        if "Max attempts exceeded" in er.message:
            print("error on create_snapshots_wait")
            print(f"Instance {Iid[0]} did not shutdown in 600 seconds")
            FailedIid.append(Iid[0])
        else:
            print("error on create_snapshots_wait_else")
            print(er.message)
            FailedIid.append(Iid[0])
```

### snapshot_copy

Now we make use of the unencrypted snapshot and create a copy turning on encryption. Again we make use of the original EBS volume tags. We call the preceding wait function again.

```python
def snapshot_copy():
    global enc_snapshot
    try:
        snap_shot = ec2.copy_snapshot(
        Description="encrypted-"+snap_prefix,
        TagSpecifications=[
                {
                'ResourceType' : 'snapshot',
                'Tags': tags_list,      
                }
            ],  
        Encrypted=True,
        SourceRegion=region,
        SourceSnapshotId=snap_check[0],
        )
        enc_snapshot = snap_shot.get('SnapshotId')
        create_snapshots_wait(snap_shot)
        if verbose:
            print(f"encrypted snapshot : {enc_snapshot}")
    except botocore.exceptions.ClientError as er:
        print("error on snapshot_copy")
        print(er.response['Error']['Message'])
        FailedIid.append(Iid[0])
    ec2.delete_snapshot(SnapshotId=unenc_snapshot,
    )
```

### create_ebs

We now make use of the encrypted snapshot to make an encrypted EBS volume

```python
def create_ebs():
    global enc_ebs_id
    try:
        enc_ebs = ec2.create_volume(
        AvailabilityZone=az2,
        Encrypted=True,
        SnapshotId=enc_snapshot,
        TagSpecifications=[
                {
                'ResourceType' : 'volume',
                'Tags': tags_list,      
                }
            ],
        )
        enc_ebs_id = enc_ebs.get('VolumeId')
        ebs_wait = enc_ebs_id
        create_ebs_wait()
    except botocore.exceptions.ClientError as er:
        print("error on create_ebs")
        print(er.message)
        FailedIid.append(Iid[0])
```

We call the previously describe waiter we used for the EBS volume detach function

### attach_new_ebs

We are as sure as we can be (without adding a further check) that the attachment is now free so we can attach the new EBS volume

```python
def attach_new_ebs():
    try:
        if verbose:
            print(f"Attaching volume {enc_ebs_id} to {volatt}")
        attach_ebs = ec2.attach_volume(
        Device=volatt,
        InstanceId=Iid[0],
        VolumeId=enc_ebs_id,
        )
    except botocore.exceptions.ClientError as er:
        print("error on attach_new_ebs")
        print(er.response['Error']['Message'])
        FailedIid.append(Iid[0])
```

### set_delete_terminate

Ensures that the EBS volumes have the value deleteonetermination set to true. Without this you can end up with a lot of orphaned EBS volumes. It's not required for volume encryption just some housekeeping.

```python
def set_delete_terminate():
    if verbose:
        print(f"deleteontermination check : {enc_ebs_id}")
    delonterm = ec2.modify_instance_attribute(
    Attribute='blockDeviceMapping',
    BlockDeviceMappings=[
    {
    'DeviceName': volatt,
    'Ebs': {
    'DeleteOnTermination': True,
    }}],
    InstanceId=Iid[0])
```

### delete_ebs

This one depends on how much confidence you have. It will terminate the original unencrypted EBS volume...gulp (You should already have a backup / snapshot lifecycle in place to give you that warm feeling)

```python
def delete_ebs():
    try:
        delete_ebs = ec2.delete_volume(
            VolumeId=volid
        )
    except botocore.exceptions.ClientError as er:
        print("error on delete_ebs")
        print(er.response['Error']['Message'])
        FailedIid.append(Iid[0])
```

### start_instance

And finally you want to start the instance back up if this was the original state.

```python
def start_instance(Iid):
    try:
        ec2.start_instances(InstanceIds=Iid)
        SuccessIid.append(Iid[0])
    except botocore.exceptions.ClientError as er:
        print("error on start_instance")
        print(er.response['Error']['Message'])
        FailedIid.append(Iid[0])
```

Conclusion
----------

There we go, a python script which retrieves AWS EC2 instances across multiple AWS Accounts assuming roles where required. The instances are checked for unencrypted EBS volumes, if found they are processed to provide the end result of encrypted EBS volumes.

The script will output a list of successful and failed instance id's for checking.

As usual, [here](https://github.com/daveihart/python-aws-ebs_encryption_public) is a link to my GitHub repo containing the source code. Feel free to ask any question or suggest any improvements.

Till next time.

Disclaimer: This script will terminate EBS volumes. Use at your own risk!