---
title: Continuous Delivery Performance Testing
tags:
  - DevOps
  - Performance Testing
  - CI / CD
categories:
  - [DevOps]
  - [Performance]
  - [Testing]
abbrlink: eadb2e78
date: 2020-04-13 22:08:58
---

# Introduction
This blog describes performance testing results of the continuous delivery (CD) workflow. Testing was performed to answer two questions:

1. Which Docker storage engine is optimal for the continuous delivery workflow?
2. Which EC2 instance types perform best in terms of price and performance?

# Test Conditions
All testing was performed with the following conditions:
- Ubuntu `15.10`
- Docker `1.10.3`
- Docker Compose `1.6.2`
- All required Docker images cached in a local registry mirror to minimise network variations
- All required Python wheels cached in a local Devpi mirror to minimise network variations
- Empty docker data volume (/var/lib/docker) as starting state for each test

Testing was performed for the internal web application. The test stage that was executed includes the following tasks:
- Pull base image
- Build development image - this includes installing OS packages, building Python wheels and installing Python wheels
- Run unit and integration tests
         
<!-- more -->

# Docker Storage Engine Comparison

The Docker storage engine comparison was performed on a single EC2 instance with the following configuration:
- c4.large instance
- 2 x 16GB SSD disks attached as instance stores configure in a software RAID0 configuration unless otherwise noted. These disks were used for Docker data storage (i.e. mounted at /var/lib/docker)
- ext4 file system used unless otherwise noted
- 30GB EBS volume attached as main OS volume

## Test Scenarios
The following table describes each scenario that was tested.

It should be noted that some storage engines are more complex to setup than others. A low complexity rating means minimal configuration is required to enable the storage engine, whereas a high complexity rating means significant configuration is required to enable the storage engine (e.g. requires separate block storage)
    
| Scenario                   |                                                                Description                                                                | Complexity |                                                                                                   More Information |
| -------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------: | :--------: | -----------------------------------------------------------------------------------------------------------------: |
| ZFS                        |                                         ZFS storage engine using software RAID0 (zfs file system)                                         |    High    | https://docs.docker.com/engine/userguide/storagedriver/zfs-driver/<br>https://wiki.ubuntu.com/Kernel/Reference/ZFS |
| ZFS (ZRAID0)               |                                    ZFS storage engine using native ZFS disk striping (zfs file system)                                    |    High    | https://docs.docker.com/engine/userguide/storagedriver/zfs-driver/<br>https://wiki.ubuntu.com/Kernel/Reference/ZFS |
| Device Mapper (direct-lvm) |                                      Device Mapper storage engine using recommended direct-lvm mode                                       |    High    |    https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/#configure-direct-lvm-mode-for-prod |
| Device Mapper (loop)       | Device Mapper storage engine using default loop mode.<br>This is the default storage engine on Ubuntu 15.10 and many Linux distributions. |    Low     |     https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/#docker-and-the-device-mapper-stor |
| BTRFS                      |                                                 BTRFS storage engine (btrfs file system)                                                  |    Higg    |                                               https://docs.docker.com/engine/userguide/storagedriver/btrfs-driver/ |
| Overlay                    |                                                          Overlay storage engine                                                           |    Low     |                                           https://docs.docker.com/engine/userguide/storagedriver/overlayfs-driver/ |

P.S.
> The original Docker storage engine, AUFS, was not considered given it is not supported in mainline Linux kernel.

## Test Results
The following chart illustrates the results:

![](https://upload-images.jianshu.io/upload_images/3113154-3a0cbe9cba880e99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Issues
Initial testing was performed using the xfs file system, however this caused a failure during Docker image building when using the Overlay storage driver. This triggered a decision to use ext4 instead as the standard file system.

# EC2 Instance Type Comparison
Testing was performed on various EC2 instance types to determine which EC2 instance types perform the best, both in terms of raw performance and price / performance.

The price / performance comparison was calculated by computing the base cost of how much each run (in terms of time taken) cost on each instance type. Additional storage costs were not included in the calculation.

## Test Scenarios
The following table describes each scenario that was tested.

| Scenario                | Instance Cost Per Hour (USD) |                                                                                                                                                                                                   Description |
| ----------------------- | :--------------------------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
| m3.medium (EBS)         |            0.093             |                                                                                                                                                                M3 Medium instance with single 30GB EBS Volume |
| r3.large (SSD)          |  0.20 + additional storage   |                                             R3 Large instance with single 30GB EBS Volume and 30GB SSD instance store volume.<br>The SSD instance store was used for the Docker data volume (/var/lib/docker) |
| t2.medium (EBS)         |             0.08             |                                                                                                                                                                T2 Medium instance with single 30GB EBS Volume |
| c3.large (SSD)          |  0.132 + additional storage  | C3 Large instance with single 30GB EBS Volume and 2x16GB SSD instance store volumes configured in software RAID0 configuration.<br>The SSD RAID0 volume was used for the Docker data volume (/var/lib/docker) |
| c4.large (EBS)          |            0.137             |                                                                                                                                                                 C4 Large instance with single 30GB EBS Volume |
| c4.large (Separate EBS) |  0.137 + additional storage  |                                                                                                                      C4 Large instance with separate 30GB EBS Volume for Docker data volume (/var/lib/docker) |
| c4.large (RAID0 EBS     |  0.137 + additional storage  |                                                                                     C4 Large instance with 2 x separate 30GB EBS Volume configured in software RAID0 for Docker data volume (/var/lib/docker) |

## Test Results (Performance)
The following chart illustrates the results in terms of performance:

![](https://upload-images.jianshu.io/upload_images/3113154-2595c65d154f95da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Test Results (Price / Performance)
The following chart illustrates the results in terms of price / performance:

![](https://upload-images.jianshu.io/upload_images/3113154-d0ab2c63ce95e49c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# Conclusions

## Storage Engine
The `overlay storage engine` appears to provide the best performance for the tested continuous delivery workflow.

Given the overlay storage engine has low complexity and is easy to configure out of the box, this is the recommended storage engine for continuous delivery workflows.

An interesting observation was the performance of DeviceMapper direct-lvm mode (this is the storage engine configuration for AWS Linux ECS-optimised instances) vs loop mode. In principle, direct-lvm mode should offer better performance but this was not observed.

## EC2 Instance Type Price / Performance
The testing clearly discounted m3.medium and r3.large instance types as not suitable for the Continuous Delivery workflow. The m3.medium instance took twice as long to execute, whilst the r3.large performance was comparable to the t2.medium performance at 2.5x the price.

Interestingly, the use of SSD instance stores did not result in any performance improvements and the best performance result was achieved on a c4.large with EBS storage.

The use of dedicated and/or multiple EBS volumes configured in a RAID0 configuration also did not result in any measurable increase in performance.

The t2.medium instance type offers the best value for money of the instance types tested, however offers burst CPU performance that can be exhausted, and the impact of running continuous delivery workflows in terms of CPU credits must be further understood.

The `c4.large` instance type offers the best performance of the instance types and was second most cost effective. This instance type is tailored towards CPU intensive workloads, offering better peak performance than the t2.medium (8 ECUs vs 6.5 ECUs) and does not incur the risk of exhausting CPU credits. The c4.large also is EBS optimised with 500Mbps dedicated EBS bandwidth and support enhanced networking (t2.medium does not support either of these enhancements).

## End-to-End Workflow Testing
Based upon `t2.medium` and `c4.large` being identified as the preferred instance types, additional testing of the workflow was conducted for these instance types. All testing was performed using the overlay storage driver.

The end-to-end workflow and results are described in the table below. Note the make tag and make publish steps were excluded as these steps are dependent on network performance to Docker Hub.

| Workflow Step          | c4.large (seconds) | t2.medium (seconds) |                                                                                                                                                                                            Description |
| ---------------------- | :----------------: | :-----------------: | -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
| make test              |        299         |         367         |          <ul><li> Pull base image</li><li> Build development image - this includes installed OS packages, building Python wheels and installing Python wheels </li><li> Run unit and integration tests |
| make database aws      |         38         |         36          |                                                                                                                                                                Mount database volume from EBS snapshot |
| make toast             |        153         |         150         |                                                                                                                                                                                  Warms database volume |
| make release           |        333         |         363         | <ul><li>Pull release environment and acceptance testing images (mysql, redis, nginx, selenium, robot) </li><li> Build release image </li><li> Start release environment </li><li> Run acceptance tests |
| Total Time             |       13m43s       |       15m16s        |                                                                                                                                                                                                        |
| Total Run Cost (cents) |        3.13        |        2.04         |                                                                                                                                                                                                        |