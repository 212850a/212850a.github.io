---
layout: post
title:  "AWS Glacier as off-site backup solution"
date:   2020-12-20 17:41:29 +0200
published: true
categories: aws backup
tags: aws backup archive duplicity glacier
---

# Overview
This instruction describes the way how to store your really important backups in encrypted format in AWS Glacier Deep Archive and pay for that really small (1$ for 1PB/month).

# Story
We all know about importance of having backups, but most of us store backups at the same room/building as data itself. 
So what would you do if you've got a thief or fire and as result you don't have your data and backup anymore?  
Correct - _off-site backup copy_.

Long time ago in addition to make a backup of data I made encrypted copies of backups and stored them off my home. It worked fine, but then I've heard about AWS Glacier and wanted to try it. And when AWS Glacier Deep Archive ([4 times cheaper](https://aws.amazon.com/s3/pricing/), for Europe(Ireland) region at time of writing that) went alive my wish was transformed into new solution for my off-site backup.

To be honest from the beginning I was not sure that the price will be as it is stated on [Amazon S3 pricing](https://aws.amazon.com/s3/pricing/), I thought that I didn't know about something more. So I've sent some test data to AWS Glacier, waited for the first bill to understand what was missed. First bill has shown that I didn't really miss anything - you don't pay for uploaded data, just for stored amount and exactly as it's stated in [Amazon S3 pricing](https://aws.amazon.com/s3/pricing/).

However there are few important things which you need to remember about:
- _Glacier metadata_ - You have to store somewhere metadata about objects which you send to Glacier (or Glacier Deep Archive), without it you won't be able to interact with them in easy way. As it's only few megabytes of data I choose to use simple S3. So I send objects to S3 but I specify DEEP_ARCHIVE as S3 Storage Class for each of them as result only metadata are left in S3 and objects themselves are transfered to Glacier Deep Archive behind the scene.
- _Restoring of data_ - Restoring of data from Glacier Deep Archive takes about 12 hours. After restoring data is available as usual S3 objects for the specified number of days (you specify it when request for restore). Restoring is the most expensive part of using of Glacier Deep Archive.
- _Minimal object life in Glacier_ - Objects that are archived to S3 Glacier and S3 Glacier Deep Archive have a minimum 90 days and 180 days of storage, respectively. It means if you sent objects to Glacier Deep Archive and delete them in a weeks you'll be charged for 180 days of storing them still.
- _S3 bucket Lifecycle rules_ - Sometimes when you upload data to Glacier Deep Archive via S3 uploads failed or canceled. The important thing - multipart parts that are already uploaded but not assembled will remain in S3 and generates cost, they will be visible on bill as GlacierStagingStorage. To avoid that I recommend to setup Lifecycle rule for whole bucket to delete incomplete multipath uploads. Not sure why it is not implemented by default. For more details - [What is AWS S3 GlacierStagingStorage Cost](https://ystatit.medium.com/what-is-aws-s3-glacierstagingstorage-cost-a0c0be216589).
- _Tax_ - I live in Vilnius, Lithuania, so VAT (21%) is added additionally to the price

# Solution
As before I continue to encrypt copies of my backups with [Diplicity](http://duplicity.nongnu.org/) to external drive, but instead of moving them physically out of home I've started to send encrypted copies to AWS Glacier Deep Archive.
As result two shell-scripts were written to automate as much as possible for these actions as they have to be done periodically (depends on your requirements), they are available in [aws-archive](https://github.com/212850a/aws-archive) repository.

# Restoration Test
When you restore some data from Glacier Deep Archive you have to take into account the following costs:
* Data Transfer OUT From Amazon S3 To Internet
	* Up to 1 GB / Month	$0.00 per GB
	* Next 9.999 TB / Month	$0.09 per GB
* Data retrievals from S3 Glacier Deep Archive
	* $0.02 per GB

It means if you want to restore 10GB of data you will pay $1,01 ($0,81 for 9GB of data transferring from S3 to you + $0,2 for 10GB of data retrievals from Glacier Deep Archive).
