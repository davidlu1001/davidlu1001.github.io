---
title: Move Terraform resources between remote states
tags: Terraform
categories: SRE
abbrlink: '40578613'
date: 2020-04-08 23:06:24
---

This document outlines the steps to move existing terraform resources to different remote states.

Reference: https://medium.com/@lynnlin827/moving-terraform-resources-states-from-one-remote-state-to-another-c76f8b76a996

### Commands

move terraform files to the new folder
run `terraform plan` to get a list of resources that *would* be deleted
Ran the commands bellow to move all of them.

```
cd infrastructure
 
terraform state mv -state-out=monitoring/.terraform/terraform.tfstate datadog_monitor.monitor-01-composite datadog_monitor.monitor-01-composite
...
 
cd monitoring
terraform state push .terraform/terraform.tfstate
...
```

Example of a `terraform plan` from `infrastructure/monitoring` folder to confirm that nothing has changed

```
If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
rm -f current.plan
rm -f *.tf.json
Nothing to do
terraform get
terraform plan -out current.plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.
 
datadog_monitor.monitor-01-composite: Refreshing state... (ID: 8413395)
...

------------------------------------------------------------------------
 
No changes. Infrastructure is up-to-date.
 
This means that Terraform did not detect any differences between your
configuration and real physical resources that exist. As a result, no
actions need to be performed.
```

`terraform plan` from main folder

```
aws_alb_listener.main: Refreshing state... (ID: arn:aws:elasticloadbalancing:us-west-2:...tion/22124b66b571c93f/1553846d70106854)
aws_alb_listener.main: Refreshing state... (ID: arn:aws:elasticloadbalancing:us-west-2:...tion/ce700936e3b458e3/4f8982a6592d6312)
aws_alb_listener.main: Refreshing state... (ID: arn:aws:elasticloadbalancing:us-west-2:...tion/cb803380f953aafc/3c695ba20ad9d237)
 
------------------------------------------------------------------------
 
No changes. Infrastructure is up-to-date.
 
This means that Terraform did not detect any differences between your
configuration and real physical resources that exist. As a result, no
actions need to be performed.
```