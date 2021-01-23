---
layout: post
title:  "AWS Glacier Deep Archive as off-site backup solution"
date:   2020-12-20 17:41:29 +0200
published: true
tags: aws backup archive duplicity glacier
---

# Overview
This instruction describes the way how to store your really important backups in encrypted format in AWS Glacier Deep Archive and pay for that really small (1$ for 1PB/month).

# Story
We all know about importance of having backups, but most of us store backups at the same room/building as data itself. 
So what would you do if you've got a thief or fire and as result you don't have your data and backup anymore?  
Correct - _off-site backup copy_.

Long time ago in addition to make a backup of data I made encrypted copies of backups and stored them off my home. It worked fine, but then I've heard about AWS Glacier and wanted to try it. And when AWS Glacier Deep Archive went live ([4 times cheaper](https://aws.amazon.com/s3/pricing/) than Glacier, for Europe(Ireland) region at time of writing that) my wish was transformed into new solution for my off-site backup.

To be honest from the beginning I was not sure that the price will be as it is stated on [Amazon S3 pricing](https://aws.amazon.com/s3/pricing/), I thought that I didn't know about something more. So I've sent some test data to AWS Glacier, waited for the first bill to understand what was missed. First bill has shown that I didn't really miss anything - you don't pay for uploaded data, just for stored amount and exactly as it's stated in [Amazon S3 pricing](https://aws.amazon.com/s3/pricing/) - $0.00099 per GB/month.

However there are few important things which final price depends on:
- _Glacier metadata_ - You have to store somewhere metadata about objects which you send to Glacier (or Glacier Deep Archive), without it you won't be able to interact with them in easy way. As it's only few megabytes of data I choose to use simple S3. So I send objects to S3 but specify DEEP_ARCHIVE as S3 Storage Class for each of them as result only metadata are left in S3 and objects themselves are transfered to Glacier Deep Archive behind the scene.
- _Restoring of data_ - Restoring of data from Glacier Deep Archive takes about 12 hours. After restoring data is available as usual S3 objects for the specified number of days (you specify it when request for restore). It means if you need to restore data - you will need to pay for that + for data which you store temporary in S3. Restoring is the most expensive part of using of Glacier Deep Archive, but it is needed only when you have real problem (you are not able to use your usual local backup).
- _Minimal object life in Glacier_ - Objects that are archived to S3 Glacier and S3 Glacier Deep Archive have a minimum 90 days and 180 days of storage, respectively. It means if you sent objects to Glacier Deep Archive and delete them in a week you'll be charged for 180 days of storing them still.
- _S3 bucket Lifecycle rules_ - Sometimes when you upload data to Glacier Deep Archive via S3 uploads failed or canceled. The important thing - multipart parts that are already uploaded but not assembled will remain in S3 and generates cost, they will be visible on bill as GlacierStagingStorage. To avoid that I recommend to setup Lifecycle rule for whole bucket to delete incomplete multipath uploads. Not sure why it is not implemented by default. For more details - [What is AWS S3 GlacierStagingStorage Cost](https://ystatit.medium.com/what-is-aws-s3-glacierstagingstorage-cost-a0c0be216589).
- _Tax_ - I live in Vilnius, Lithuania, so VAT (21%) is added additionally to the price

# Solution
As before I continue to encrypt copies of my backups with [Diplicity](http://duplicity.nongnu.org/) to external drive, but instead of moving it physically out of home I've started to send encrypted copies to AWS Glacier Deep Archive.
As result two shell-scripts were written to automate as much as possible for these actions as they have to be done periodically (depends on your requirements), they are available in [aws-archive](https://github.com/212850a/aws-archive) repository with all required instructions.

# Restoration Test
To restore some data from Glacier I did the following:
- Initiate restore (from S3 web GUI) for all objects from desired folder (Media/2020)
- Provide getobject right for S3 objects from bucket for user which I used in aws cli. I did it by creating additional policy and assigning it to desired user.
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "s3:GetObject",
            "Resource": [
                "arn:aws:s3:::my-deep-archive/*"
            ],
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": "123.45.67.89"
                }
            }
        }
    ]
}
```
- Wait until restore is completed (from GUI, typically 12 hours). After object is restored you will get something like:
```
$ aws s3api head-object --bucket my-deep-archive --key Media/2020/duplicity-full-signatures.20201105T145850Z.sigtar.gpg
-------------------------------------------------------------------------------------------
|                                       HeadObject                                        |
+---------------+-------------------------------------------------------------------------+
|  AcceptRanges |  bytes                                                                  |
|  ContentLength|  7362508                                                                |
|  ContentType  |  binary/octet-stream                                                    |
|  ETag         |  "921c8ba89fa161f30525c8380196c130"                                     |
|  LastModified |  Thu, 05 Nov 2020 15:43:48 GMT                                          |
|  Restore      |  ongoing-request="false", expiry-date="Mon, 21 Dec 2020 00:00:00 GMT"   |
|  StorageClass |  DEEP_ARCHIVE                                                           |
+---------------+-------------------------------------------------------------------------+
```
As you may see ongoing-request="false" - it means object is restored and it will be stored until 2020/12/21, so you have to download it from S3 before that otherwise it will be removed and additional restore will be required. So plan in advanced how much time you need to download restored objects from S3.
- To use --recursive with --force-glacier-transfer key to download all objects from desired folder. If you use just --recursive key you'll get warning `Object is of storage class GLACIER. Unable to perform download operations on GLACIER objects. You must restore the object to be able to perform the operation. See aws s3 download help for additional parameter options to ignore or force these transfers.` and folder won't be recursively. When using the --force-glacier-transfer flag it doesn't do another restore, it just ignores the API saying the object is in Glacier and tries anyways. It will fail if the object is not restored (it won't try to restore it).

```
$ aws s3 cp --recursive --force-glacier-transfer s3://my-deep-archive/Media/2020/ 2020/
download: s3://my-deep-archive/Media/2020/duplicity-full.20201105T145850Z.manifest.gpg to 2020/duplicity-full.20201105T145850Z.manifest.gpg
download: s3://my-deep-archive/Media/2020/duplicity-full-signatures.20201105T145850Z.sigtar.gpg to 2020/duplicity-full-signatures.20201105T145850Z.sigtar.gpg
download: s3://my-deep-archive/Media/2020/duplicity-full.20201105T145850Z.vol2.difftar.gpg to 2020/duplicity-full.20201105T145850Z.vol2.difftar.gpg
download: s3://my-deep-archive/Media/2020/duplicity-full.20201105T145850Z.vol1.difftar.gpg to 2020/duplicity-full.20201105T145850Z.vol1.difftar.gpg
$ ls -l 2020
total 1118024
-rw-r--r-- 1 root root        369 Nov  5 17:43 duplicity-full.20201105T145850Z.manifest.gpg
-rw-r--r-- 1 root root 1073779354 Nov  5 17:43 duplicity-full.20201105T145850Z.vol1.difftar.gpg
-rw-r--r-- 1 root root   63695756 Nov  5 17:43 duplicity-full.20201105T145850Z.vol2.difftar.gpg
-rw-r--r-- 1 root root    7362508 Nov  5 17:43 duplicity-full-signatures.20201105T145850Z.sigtar.gpg

```

## Restore costs
When you restore some data from Glacier Deep Archive you have to take into account the following costs:
* Data retrievals from S3 Glacier Deep Archive
	* $0.02 per GB 
* S3 Standard General purpose storage - to store restored data temporary
	* First 50 TB / Month - $0.023 per GB
* Data Transfer OUT From Amazon S3 To Internet
	* Up to 1 GB / Month - $0.00 per GB
	* Next 9.999 TB / Month	- $0.09 per GB


It means if you want:
* to restore 10GB of data and store them for 1 week in S3 you will pay $1.063 (excluded tax):
	* $0,2 for 10GB of data retrievals from Glacier Deep Archive
	* $0.053 for storing 10GB of data in S3 for 7 days
	* $0.81 for 9GB of data transferring from S3 to you
* to restore 100GB of data and store them for one month in S3 you will pay $13.21 (excluded tax):
	* $2 for 100GB of data retrievals from Glacier Deep Archive
	* $2.3 for storing 100GB of data in S3 for 1 month
	* $8.91 for 99GB of data transferring from S3 to you

I would say it's really good price for assurance and ability to restore really important for you data.

