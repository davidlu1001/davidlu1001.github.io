---
title: AWS Glue - Knowledge Base
tags:
  - AWS
  - Glue
  - ElasticSearch
  - AppSync
categories:
  - [AWS]
  - [SRE]
  - [Glue]
  - [ElasticSearch]
abbrlink: 56cc7bd6
date: 2018-08-20 23:41:58
---

# Glue - Official FAQ

The official doc for troubleshooting could be found [here](https://docs.aws.amazon.com/glue/latest/dg/troubleshooting-glue.html)

# Troubleshooting - Notes & Tips

Notes and tips for Glue when implementing the ETL process:

## Naming rules / conventions for AWS services

- S3 bucket name can either NOT uppercase nor NOT contain "_" 
- Dynamo DB table / Name can NOT contain "-"
- ES / Name: can NOT contain "-"

## ACM SSL Cert

Using `us-east-1` region for AWS CloudFront (certificate)

## ElasticSearch Schema

Updated the ES mappings in all environments so that field A searches are now case insensitive and will work with spaces in A names

e.g. Fixed the issue for field A so searches are now case insensitive and will work with spaces in A names

```
query='{"query":{"terms":{"name":["gift vouchers"]}}}'
python sign_request.py -X GET $es/A/_search -d "$query"
| jq -r '.hits.hits'
[
  {
    "_index": "A",
    "_type": "A",
    "_id": "AWSXXXXXXXXXXXXXXXX",
    "_score": Y.Z,
    "_source": {
      "name": "Gift Vouchers"
    }
  }
]
```


e.g. Fixed the issue for field B with spaces not returning quotes:

```
{
  "settings": {
    "analysis": {
      "normalizer": {
        "lowercase_normalizer": {
          "type": "custom",
          "char_filter": [],
          "filter": [
            "lowercase",
            "asciifolding"
          ]
        }
      }
    }
  },
  "mappings": {
    "A": {
      "properties": {
        "name": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256,
              "normalizer": "lowercase_normalizer"
            }
          }
        }
      }
    }
  }
}
```

## Publish Glue scripts to S3

AWS S3 wildcards for copying scripts to bucket:

```
aws s3 sync . s3://$(S3_BUCKET)/$(FUNCTION_NAME)_Glue/ --recursive --exclude "*" --include "glue_*.py"
```

## Glue - Crawler - *DependsOn*

If not using `DependsOn: Connection` in Glue Crawler, it won't create Resource `Connection` before `Crawler`.

And the error message is as follows:

```
===> JDBC Connection not registered: (Service: AWSGlue; Status Code:
400; Error Code: InvalidInputException; Request ID:
5cd88624-78f3-11e8-bf94-23377d56a892)
```

Code in `AWS CloudFormation` template:
```
# Create a crawler to crawl the flights data
  CrawlerFlights:
    Type: AWS::Glue::Crawler
    DependsOn: GlueConnectionRDS
```

## ES Timeout issue - using NAT gateway

### Background

ES use Internet Endpoint (not VPC Endpoint)
Can’t use `VPC endpoint` for search services - it must be accessible from `AWS AppSync`
So when creating Glue connection, it would connect to DB

### AWS Official Doc

> All JDBC data stores that are accessed by the job must be available from the VPC subnet.
>
> If your job needs to access both VPC resources and the public internet, the VPC needs to have a Network Address Translation (NAT) gateway inside the VPC.
>
> The network interface is not assigned any public IP addresses. AWS Glue requires internet access (for example, to access AWS services that don't have VPC endpoints). You can configure a network address translation (NAT) instance inside your VPC, or you can use the Amazon VPC NAT gateway.

### Adding Glue Subnet

So a specific Glue subnet with public access is needed.

### DB Security Group

For the DB need to add `self ref SG`, otherwise it'll get ERROR as follows:
```
Error: Inbound Rule in Security Group Required
```

[AWS Doc](https://docs.aws.amazon.com/glue/latest/dg/glue-troubleshooting-errors.html#error-inbound-self-reference-rule):
> At least one security group must open all ingress ports. To limit traffic, the source security group in your inbound rule can be restricted to the same security group.

### Code Example (Security Group)

```
ADBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription:
        Fn::Sub: ${AWS::StackName} A DB Security Group
      VpcId:
        Fn::ImportValue:
          Fn::Sub: ${Environment}-vpc-id
      SecurityGroupIngress:
        - CidrIp:
            Fn::ImportValue:
              Fn::Sub: ${Environment}-database-subnets-cidr
          FromPort: 0
          ToPort: 65535
          IpProtocol: tcp
      Tags:
        - Key: Name
          Value:
            Fn::Sub:
${AWS::StackName}-glue-A-primary-security-group
  ASecurityGroupIngress:
    Type: AWS::EC2::SecurityGroupIngress
    DependsOn: ADBSecurityGroup
    Properties:
      GroupId:
        Fn::Sub: ${ADBSecurityGroup.GroupId}
      IpProtocol: tcp
      FromPort: 0
      ToPort: 65535
      SourceSecurityGroupId:
        Fn::Sub: ${ADBSecurityGroup.GroupId}
```

# Code Example (Glue Cloudformation Template)

```
# Glue Job
  XGlueJob:
    Type: AWS::Glue::Job
    Properties:
      Name:
        Fn::Sub: ${AWS::StackName}-X-GlueJob
      #LogUri: "wikiData" 
      Connections:
        # https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-glue-job-connectionslist.html
        Connections:
          - Ref: XGlueConnectionRDS
      Role:
        Ref: GlueJobTriggerRole
      Command:
        # The name of the job command: this must be "glueetl"
        # https://docs.aws.amazon.com/glue/latest/dg/aws-glue-api-jobs-job.html#aws-glue-api-jobs-job-JobCommand
        Name: glueetl
        ScriptLocation:
          Fn::Sub:
s3://${S3Bucket}/productServicesGlue/glue.py
      DefaultArguments:
        "--continuation-option": "continuation-enabled"
        # use "TempDir" refer to:
        # https://docs.aws.amazon.com/glue/latest/dg/populate-with-cloudformation-templates.html#sample-cfn-template-job-jdbc
        "--TempDir":
s3://aws-glue-temporary-{account_id}-${region}/admin
        "--extra-py-files": 
          Fn::Sub:
s3://${S3Bucket}/productServicesGlue/productServicesGlue.zip
        "--elasticsearch_host":
          Ref: ElasticSearchHost
        "--db_name":
          Ref: XGlueDatabaseName
        "--table_name":
          Fn::Join:
          - ''
          - - Ref: XTablePrefixName
            - Ref: XGlueTableName
      MaxRetries: 0
      AllocatedCapacity: 2  
      ExecutionProperty:   
        MaxConcurrentRuns: 1
```

# Reference

- https://docs.aws.amazon.com/AmazonS3/latest/dev/BucketRestrictions.html
- https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.NamingRulesDataTypes.html
- https://docs.aws.amazon.com/cli/latest/reference/s3/index.html#use-of-exclude-and-include-filters
- https://docs.aws.amazon.com/glue/latest/dg/glue-troubleshooting-errors.html#error-inbound-self-reference-rule