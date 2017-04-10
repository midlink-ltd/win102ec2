# Downloading the image

Download the windows image you want.

AWS vmimport supported versions:
Microsoft Windows 10 (Professional, Enterprise, Education) (US English) (64-bit only)

So Home wont work.

You can download the trial Enterprise trial here: https://www.microsoft.com/en-us/evalcenter/evaluate-windows-10-enterprise

# Creating the virtual machine

Use virtualbox to create a new virtual machine, make sure that it uses the VHD format.
Install the Windows Image onto it.
Install teamviewer on the virtual machine or enable remote desktop.
Exit the virtual machine.

# Install and configure awscli
```bash
sudo apt install awscli
aws configure
````

http://docs.aws.amazon.com/general/latest/gr/aws-access-keys-best-practices.html
During configure you can add your:

AWS access key.
AWS secret access key.
Default region.

If you set a default region you dont have to specify the region parameter in the following commands.

# Create an S3 bucket

The bucketname must be unique.

````bash
aws s3 mb s3://bucketname --region eu-central-1
````

# Upload image to s3
Move to the folder you store the virtual machine file and upload the virtual image to the s3 bucket.

````bash
cd myvmfolder
aws s3 cp myimage.vhd s3://bucketname --region eu-central-1
````

# Configuration files

Create a trust policy in the file trust-policy.json

```json
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Principal": { "Service": "vmie.amazonaws.com" },
         "Action": "sts:AssumeRole",
         "Condition": {
            "StringEquals":{
               "sts:Externalid": "vmimport"
            }
         }
      }
   ]
}
````

Create a vmimport role and add vim import/export access to it.

````bash
aws iam create-role --role-name vmimport --assume-role-policy-document file://trust-policy.json
````

Create a file named role-policy.json replace the !!REPLACEME!! to the bucketname you are using.

````json
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Action": [
            "s3:ListBucket",
            "s3:GetBucketLocation"
         ],
         "Resource": [
            "arn:aws:s3:::!!REPLACEME!!"
         ]
      },
      {
         "Effect": "Allow",
         "Action": [
            "s3:GetObject"
         ],
         "Resource": [
            "arn:aws:s3:::!!REPLACEME!!/*"
         ]
      },
      {
         "Effect": "Allow",
         "Action":[
            "ec2:ModifySnapshotAttribute",
            "ec2:CopySnapshot",
            "ec2:RegisterImage",
            "ec2:Describe*"
         ],
         "Resource": "*"
      }
   ]
}
````

Add the policy to the vmimport role.

````bash
aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document file://role-policy.json
````

Create a configuration file on your computer called containers.json.
Replace bucketname and myimage.vhd with your bucket and image name.

````json
[{ "Description": "Windows 10 Base Install", "Format": "vhd", "UserBucket": { "S3Bucket": "bucketname", "S3Key": "myimage.vhd" } }]
````

Start to import from S3 to Ec2.

````bash
aws ec2 import-image --description "Windows 2008 OVA" --disk-containers file://containers.json
````

This may take a while you can check on the status of the import.

````bash
aws ec2 describe-import-image-tasks --region eu-central-1
````

After the AMI is created you can navigate to the EC2 console, select the correct region.
