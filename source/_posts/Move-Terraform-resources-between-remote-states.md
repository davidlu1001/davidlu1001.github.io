---
title: Move Terraform resources between remote states
tags: 
  - Terraform
  - SRE
categories:
  - [SRE]
  - [Terraform]
abbrlink: '40578613'
date: 2019-04-28 23:06:24
---

This document outlines the steps to move existing terraform resources to different remote states.

Reference: https://medium.com/@lynnlin827/moving-terraform-resources-states-from-one-remote-state-to-another-c76f8b76a996

### Steps

#### 1. Move Terraform files to the new folder

#### 2. Run make plan to get a list of resources that *would* be deleted

sample to generate desired commands used for next step

```
terraform plan | grep '\s-\s' | sed -e 's/\s-\s//g' | awk '{print "terraform state mv -state-out=monitoring/.terraform/terraform.tfstate "$1,$1}'
```

<!-- more -->

#### 3. Ran commands bellow to move

```
cd infrastructure
 
terraform state mv -state-out=monitoring/.terraform/terraform.tfstate datadog_monitor.monitor-01-composite datadog_monitor.monitor-01-composite
...
```
 
#### 4. Push remote state in new folder

```
cd monitoring

terraform state push .terraform/terraform.tfstate
```

As in the company we use `Makefile` to chain several steps together, so remember to push the remote state before run `make plan` (otherwise the tfstate will be deleted):

```
clean:
	rm -rf .terraform
	rm -f current.plan
	rm -f *.tf.json

plan: clean check_remote_modules init get_modules
	terraform plan -out current.plan ${PLAN_OPTIONS}
```

#### 5. Check by make plan

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

#### 6. Recovery in case ruin the remote state

For Step 4, if ran `make plan` before pushing the remote state in new folder, the dedicated remote state file will be deleted, and the make plan output will show that the new resources would be created.

So for recovery, there're two places that can give some hints.

##### Local terraform backup
Terraform usually creates a backup file under `.terraform/` (after run `terraform state mv` command in Step 3)

```
-rw-r--r-- 1 davidlu 329622 May  5 10:46 terraform.tfstate.1588632389.backup
-rw-r--r-- 1 davidlu 328503 May  5 10:46 terraform.tfstate.1588632399.backup
-rw-r--r-- 1 davidlu 327160 May  5 10:46 terraform.tfstate.1588632409.backup
-rw-r--r-- 1 davidlu 326185 May  5 10:47 terraform.tfstate.1588632420.backup
-rw-r--r-- 1 davidlu 322535 May  5 10:47 terraform.tfstate.1588632429.backup
-rw-r--r-- 1 davidlu 319527 May  5 10:47 terraform.tfstate.1588632438.backup
-rw-r--r-- 1 davidlu 316520 May  5 10:47 terraform.tfstate.1588632449.backup
```

##### S3 bucket (recommend)
We can still restore an older remote state version from S3

***Steps***:

1. Download original older remote state file from S3 (e.g. `production-terraform/datastores/production`)

2. Extract the related resources (e.g. blaze)
   -  Search the keyword and copy & paste
   -  The above process can be tedious and error-prone: introduce [gron](https://github.com/tomnomnom/gron) here - would be helpful for **double-check** (can make JSON greppable and turn filtered data back into JSON)

```
e.g.

gron original_older_remote_state.json | grep blaze | gron -u
```

3. Tweak config to match

- Base the previous steps, we've got the remote state file in new folder, something like this:

```
{
    "version": 3,
    "terraform_version": "0.11.14",
    "serial": 3,
    "lineage": "e092833e-c86c-edb2-1c15-c5e04f1923d7",
    "modules": [
        {
            "path": [
                "root"
            ],
            "outputs": {},
            "resources": {

                [...RESOURCES...]

            },
            "depends_on": []
        }
    ]
}
```

  - Download remote state file for new folder (e.g. `production-terraform/datastores/services/blaze`)

```
{
    "version": 3,
    "serial": 2,
    "lineage": "e092833e-c86c-edb2-1c15-c5e04f1923d7",
    "backend": {
        "type": "s3",
        "config": {
            "bucket": "production-terraform",
            "key": "datastores/services/blaze",
            "region": "us-west-2"
        },
        "hash": 3065110275131057704
    },
    "modules": [
        {
            "path": [
                "root"
            ],
            "outputs": {},
            "resources": {},
            "depends_on": []
        }
    ]
}
```


   - Make sure has the same lineage, and higher version of serial (compared with destination in S3)

> ***Differing lineage***: The "lineage" is a unique ID assigned to a state when it is created. If a lineage is different, then it means the states were created at different times and its very likely you're modifying a different state. Terraform will not allow this.
>
> ***Higher serial***: Every state has a monotonically increasing "serial" number. If the destination state has a higher serial, Terraform will not allow you to write it since it means that changes have occurred since the state you're attempting to write.
>
> Ref: https://www.terraform.io/docs/backends/state.html

   - Upload new tfstate file to S3

```
e.g.

aws s3 cp blaze s3://production-terraform/datastores/services/blaze
```

  - Run `terraform plan` in new folder (no changes expected)