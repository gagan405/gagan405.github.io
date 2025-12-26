+++
title = "Oracle Cloud WTF"
date = "2025-12-26"

[taxonomies]
tags=["oci", "oracle-cloud", "wtf"]
+++

## First impressions with OCI

I am surprised and utterly disappointed with how OCI cannot handle super basic stuff.
Here is what I did:

1. Create a free tier instance
2. ssh to the instance
3. try installing openjdk, or just `yum update`
```shell
sudo yum install java-11-openjdk -y
```
4. that hangs
5. can't ssh to the same instance anymore

This, while there has been no changes in security rules! And I was able to login just 5 minutes ago.

Here is a screen recording of this behavior: https://youtu.be/tnDdaiCZh10

No wonder nobody uses this!

Also, dear OCI, who signed off on this UI color combination and aesthetics?