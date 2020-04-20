---
title: Cost Optimization cases on AWS
tags:
  - Cost Optimization
  - AWS
  - SRE
categories:
  - - SRE
  - - AWS
abbrlink: 79eee546
date: 2020-04-20 21:08:44
---

Rather than write a big, manual-style cost optimization guide, I'd like to share a few pits I've encountered during the whole process. 

# Tools

Common tools for AWS cost optimization are as follows:

1. AWS Cost Explorer
2. Cost Reports in S3
3. AWS Trusted Advisor - Cost Optimization

# Strategy

Based on `AWS Cost Optimization Best Practice`, the main measures are probably the following aspects:

1. **Right Sizing**: Use a more appropriate (convenient) Instance Type / Family (for EC2 / RDS) 
2. **Price models**: leverage Reserved Instances (RI) and Spot Instances (SI)
3. **Delete / Stop unused resources**: e.g. EBS Volume / Snapshot, EC2 / RDS / EIP / ELB, etc.
4. **Storage Tier / Backup Policy**: Move cold data to cheaper storage tiers like Glacier; Review EBS Snapshot / RDS backup policy
5. **Right Tagging**: Enforce allocation tagging, while improving Tag coverage and accuracy
6. **Scheduling On / Off times**: Review existing Auto Scaling policies; Stop instances used in Dev and Prod when not in use and start them again when needed

Corresponding to the specific situation of the company, mainly want to talk about 3 and 4 mentioned above.

# About Backup

## AWS Backup

At first, the company used Lambda Function (Python script) to handle EBS / RDS backup. Later, because of the release of AWS Backup Service, it was decided to migrate backup management to AWS Backup.

However, for Database backup, because AWS Backup Service does not support `Aurora`, so we still need to keep the original Lambda.

> AWS Backup currently supports all Amazon RDS database engines except Amazon Aurora.

For AWS Backup, it can support multiple AWS Services / Resources, such as:

* [Amazon EFS](https://docs.aws.amazon.com/efs/latest/ug/awsbackup.html)

* [DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/BackupRestore.html)

* [Amazon EBS Snapshots](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSSnapshots.html)

* [Amazon RDS DB Instances](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_CommonTasks.BackupRestore.html)

* [AWS Storage Gateway](https://docs.aws.amazon.com/storagegateway/latest/userguide/backing-up-volumes.html#backup-volumes-cryo)

Unfortunately, AWS Backup does not support the `Wildcard (*)` to match resources.

For example:

Internally we use `Terraform` as IaC ([Infrastructure as code](https://en.wikipedia.org/wiki/Infrastructure_as_code)) to automate the management of resources on AWS.

Examples of corresponding AWS Backup codes:

```
  resources                       = [
    # RDS
    "arn:aws:rds:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:db:redash",
    # DynamoDB
    "arn:aws:dynamodb:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:table/crm",
    # EBS
    "arn:aws:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:volume/
vol-054cfe133afc1dc10",
    # EFS
    "arn:aws:elasticfilesystem:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:file-system/${filesystem-id}",
    # StorageGateway
    "arn:aws:storagegateway:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:gateway/${gateway-id}/volume/${volume-id}",
  ]

  tags = [
  {
    type  = "STRINGEQUALS"
    key   = "Environment"
    value = "Production"
  },
  {
    type  = "STRINGEQUALS"
    key   = "Cost Center"
    value = "Platform"
  },
  {
    type  = "STRINGEQUALS"
    key   = "MakeSnapshot"
    value = "True"
  },
  ]
```

Therefore, the corresponding Backup solution is:

- Either specify the specific `Resource ID` (use `resources` to specify, but cannot support something like `arn:aws:rds:${region}:${account_id}:db:*`)

- Either use `Tags` to mark the resources to be backed up by AWS Backup Service

Of course, the former is certainly not realistic, which means:

***The feature of AWS Backup determins (well, I believe this is not a bug), in fact you can only use `Tags`, and need to set `Resources` to empty `[]` (by default it's for all Resource types supported by AWS Backup). Thus laying a hidden danger.***

> Note: The relationship between `Tags` is `OR`, so as long as  one `Tag` get matched, AWS Backup will back it up.

The foreplay is over, and the pit is coming!

At the beginning of March, it was found that the AWS bill was obviously abnormal. After analysis, it was found that the cost of EBS snapshots increased too fast.

![](https://raw.githubusercontent.com/davidlu1001/davidlu1001.github.io/hexo/uPic/02_ebs_cost.png)

As you can see in the chart, it started to grow slowly from around `January 15th`, so what happened at this time?

The internal search was fruitless. After googling, I found a suspicious [What's New](https://aws.amazon.com/about-aws/whats-new/2020/01/aws-backup-adds-support-amazon-elastic-cloud-compute-instance-backup/) message from AWS official website.

Hmmmmmm, perfectly matched...

![](https://raw.githubusercontent.com/davidlu1001/davidlu1001.github.io/hexo/uPic/03_aws_what_new.png)

Okay! Looks like AWS help me to back up all the EC2 instances by default... Thank you!

According to the documentation, for AWS Backup, in addition to EBS Snapshot itself, it will also make an AMI copy for each EC2 instance, so that's why there are a lot of AMIs.

> When backing up an Amazon EC2 instance, AWS Backup takes a snapshot of the root Amazon EBS storage volume, the launch configurations, and all associated EBS volumes. AWS Backup stores certain configuration parameters of the EC2 instance, including instance type, security groups, Amazon VPC, monitoring configuration, and tags. The backup data is stored as an Amazon EBS volume-backed AMI (Amazon Machine Image).

![](https://raw.githubusercontent.com/davidlu1001/davidlu1001.github.io/hexo/uPic/04_aws_backup_ami.png)

![](https://raw.githubusercontent.com/davidlu1001/davidlu1001.github.io/hexo/uPic/05_aws_backup_ebs.png)

According to the previous AWS Backup execution logic, for all supported types (including newly supported EC2 on January 13), as long as the tag is matched, the backup will be performed.

So, for EC2 AMI / EBS snapshot, how Tags are added by default?

Checked the `Terraform` code:

```
resource "aws_autoscaling_group" "main" {
...
  tag {
    key                 = "Name"
    value               = "Value"
    propagate_at_launch = true
  }
...
}
```

Since `propagate_at_launch = true` is used, the tags of ASG will be copied when EC2 launches.

In addition, after EC2 is started, the initialization script will be run through Puppet to copy the ASG / EC2 tags to EBS Volumes. Code example as follows:

```
# Retrieve the tags attached to the instance if no auto-scaling group exists.
# Add all tags retrieved from the auto-scaling group to all EBS volumes attached to the current EC2 instance.

if [ "$asg" != "" ]; then
    aws ec2 create-tags --region "${region_id}" --resources ${volumes} --tags "${tags}"
else
    tags=$(aws ec2 describe-tags --region "${region_id}" --filters "Name=resource-id,Values="${instance_id}"" --query 'Tags[*].{Key:Key,Value:Value}')
    aws ec2 create-tags --region "${region_id}" --resources ${volumes} --tags "${tags}"
fi
```

So the current Tags replication chain:

> ASG-> EC2-> EBS Volume

As for the solution, just to differentiate `Tags` between EC2 and EBS, and delete the tags that will be matched by AWS Backup (here is `MakeSnapshot*`), to achieve the purpose of "Backing up EBS only, not EC2 instance":

```
if [ "$asg" != "" ]; then
    aws ec2 create-tags --region "${region_id}" --resources ${volumes} --tags "${tags}"
    aws ec2 delete-tags --resources "${instance_id}" --tags Key=MakeSnapshot Key=MakeSnapshotShortTerm Key=MakeSnapshotLongTerm
else
    tags=$(aws ec2 describe-tags --region "${region_id}" --filters "Name=resource-id,Values="${instance_id}"" --query 'Tags[*].{Key:Key,Value:Value}')
    aws ec2 create-tags --region "${region_id}" --resources ${volumes} --tags "${tags}"
    aws ec2 delete-tags --resources "${instance_id}" --tags Key=MakeSnapshot Key=MakeSnapshotShortTerm Key=MakeSnapshotLongTerm
fi
```

The effect after cleaning:

![](https://raw.githubusercontent.com/davidlu1001/davidlu1001.github.io/hexo/uPic/06_cost_ebs_after.png)

After this battle, I completely felt the fear of being dominated by AWS Bills :-(

You can't plant heels in the same place, so just add monitoring for news from AWS :-)

![](https://raw.githubusercontent.com/davidlu1001/davidlu1001.github.io/hexo/uPic/07_monitor%20plugin.png)

# About DB storage

**RDS backup storage excess free allocation**

> Potential increase to your Amazon Aurora bill [AWS Account: XXXXXXXXX]
>
> Starting March 1, 2020, we will start charging for backup storage in excess of the free allocation. There will not be be any back-dated or retroactive charges for use of Aurora backups before March 1, 2020.
>
> You can review your current backup retention policy and snapshots on the RDS Management Console. You can lower your monthly bill by reducing your backup retention window or by deleting unnecessary snapshots.

Usually, in our SRE team who is on duty will also pay attention to daily BAU such as consultation / email etc., but on that day, the unlucky guy was too busy to ignore the AWS email when received it, and therefore didn’t create a JIRA ticket, but (the interesting thing is) everyone in the team missed it as well, which is pretty strange.

As a result, we didn't realize until reviewing the AWS Cost, and the price is not cheap actually:

> $ 0.095 per additional GB-month of backup storage exceeding free allocation

Currently the company's default setting of `backup_retention_period` for RDS is` 30` days, and this time, of course, we need to reduce it.

The effect is as follows, better than nothing:

![](https://raw.githubusercontent.com/davidlu1001/davidlu1001.github.io/hexo/uPic/08_cost_rds.png)

## About Data Transfer

One of the more interesting things during this period is that, I found there is something called ***EC2-others*** in the AWS Cost Report. Here is the official statement: 

> The [**EC2-Other**](https://aws.amazon.com/blogs/aws-cost-management/tips-and-tricks-for-exploring-your-data-in-aws-cost-explorer-part-2/) category includes multiple service-related usage types, tracking costs associated:
>
> * Amazon EBS volumes and snapshots
> * Elastic IP addresses
> * NAT gateways
> * Data transfer

Through ithe Billing Report, I found that the following is the culprit:

![](https://raw.githubusercontent.com/davidlu1001/davidlu1001.github.io/hexo/uPic/09_cost_datatransfer.png)

### How to locate

Using `Cost Explorer`, we can locate the specific item with the following settings:

![](https://raw.githubusercontent.com/davidlu1001/davidlu1001.github.io/hexo/uPic/10_ce_locate_dt.png)

Finally, the major part of Data Transfer is located, which comes from a (business) Load Balancer, followed by the Network Load Balancer (NLB) that for Logging.

### How to reduce data transmission costs

A picture is worth a thousand words:

![](https://raw.githubusercontent.com/davidlu1001/davidlu1001.github.io/hexo/uPic/11_dt_cost.png)

There're some general strategies, such as:

- Limit data transfer to the same AZ as much as possible, or between AZs or the same Region.

- Use Private IP whenever possible.

For example:

- EC2 that needs to communicate in a development or test environment should be in the same Availability Zone (AZ) to avoid data transfer costs.

- EC2 can use CloudFront (CDN) if it needs to transfer data such as pictures / videos to Public.

- For resources in different Regions or multiple Accounts, can use [VPC Peering](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html) or [VPC Sharing](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-sharing.html) to further reduce transfer costs.

In reality, a multi-pronged strategy has been adopted.

After locating certain LB problems, the method of "directly reducing the transmitted data" was adopted, e.g.: 

- Turn off `log_subrequest` config in `OpenResty(Nginx)`

    > disables logging of subrequests into [access_log](http://nginx.org/en/docs/http/ngx_http_log_module.html#access_log)

- Add filter strategy in `Logstash` to discard some useless data

- In `Rsyslog`, enable the compression mode for [omfwd module](https://rsyslog.readthedocs.io/en/latest/configuration/modules/omfwd.html). Generally, a medium compression level should be fine, for example, `ZipLevel` can be set to `3-5`, and `compression.mode` can be set to `single`, which means each message will be evaluated and only the larger message will be compressed. And here in order to save costs, a more aggressive strategy is adopted, which will introduce delay but it's acceptable.

    ```
    ZipLevel = "9"
    compression.mode = "stream: always"
    compression.stream.flushOnTXEnd = "off"
    ```

- In addition, can also consider adjusting the `index.codec` configuration of `ElasticSearch`, using `best_compression` to replace the default compression algorithm `LZ4` to achieve a higher compression ratio, in order to reduce data transfer across AZ (such as `shard allocation`)

Well, for the time being, it is predicted that `Cost Optimization` will be a protracted battle, especially in the current environment of global economic impact.

Therefore, we really need to pay more attention to cost optimization, not only from the beginning of architecture / solution design, development and daily maintenance, but also need to create a lean cost-centric culture.

# References

- [AWS Cost Management](https://aws.amazon.com/aws-cost-management/)

- [Cost Optimization-AWS Well-Architected Framework](https://wa.aws.amazon.com/wat.pillar.costOptimization.en.html)

- [AWS Backup: working-with-other-services](https://docs.aws.amazon.com/aws-backup/latest/devguide/working-with-other-services.html)