---
title: Amazon S3
date: 2023-10-02 23:36:23
tags:
- S3
- ★☆★
---

# Introduction
*<font color=royalblue>Infinitely Scaling Storage</font>*



# Amazon S3 - Components
Amazon S3 allows people to store objects (files) in buckets (directories).

## Buckets
*<font color=red>S3 looks like a global service but buckets are created in a region</font>*

- Buckets must have a globally unique name (across all regions all accounts)
- Buckets are defined at the region level
- Name Convention
  - No uppercase, No underscore
  - 3-63 characters long
  - Not an IP
  - Must start with lowercase letter or number
  - Must NOT start with the prefix `xn--`
  - Must NOT end with the suffix `-s3alias`


## Objects
*<font color=red>Objects (files) have a Key. Key with very long names that contain slashes ("/").</font>*
*<font color=red>There is no concept of "directories" within buckets</font>*

- The key is the FULL path:
  - s3://my-bucket/my_folder/another_folder/my_file.txt
- The key is composed of <font color=red>prefix</font> + <font color=green>object name</font>
  - s3://my-bucket/<font color=red>my_folder/another_folder/</font><font color=green>my_file.txt</font>
- Object values are the content of the body:
  - Max. Object Size is 5 TB (5000 GB)
  - If uploading more than 5 GB, must use "multi-part upload".
- Metadata
  - List of text key/value pairs
  - System or user metadata
- Tags
  - Unicode key/value pair
  - Up to 10
  - Useful for security/lifecycle
- Version ID
  - If versioning is enabled


# Use Cases
- Backup & Storage
- Disaster Recovery
- Archive
- Hybrid Cloud Storage
- Application/Media Hosting
- Data Lakes & Big Data Analytics
- Software Delivery
- Static Website

## Business Use Cases
- Nasdaq stores 7 years of data into Glacier
- Sysco runs analytics on its data and gain business insights


# Limitations
