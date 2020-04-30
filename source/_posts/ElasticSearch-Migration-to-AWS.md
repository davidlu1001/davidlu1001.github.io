---
title: ElasticSearch migrated from EC2 to AWS managed ES
tags:
  - ElasticSearch
  - SRE
  - AWS
categories:
  - - SRE
  - - AWS
  - [ElasticSearch]
abbrlink: fdd4d29b
date: 2019-11-12 21:07:58
---

# Background
Let's say there's a legacy platform called XXX, and it still poses an operational cost for SRE to manage the monitoring / alerting / upgrade etc.

At present, XXX's ElasticSearch architecture consists of 3 hosts (m4.xlarge) acting as master, client and data nodes, for XXX's searching functionality.

All of them are using Ubuntu Trusty (14.04) image, which is no longer supported and need to be migrated to Xenial (16.04). Its usage seems pretty low and cost wise so we would like to migrate the XXX ES to AWS managed ElasticSearch Service to reduce operational overhead.

This doc will cover the project plan / checklist for migration.

# Solution overview
Elasticsearch (ES) indexes can be migrated with following steps:

- Create baseline indexes
  - Create a snapshot repository and associate it to an AWS S3 Bucket.
  - Create the first snapshot of the indexes to be migrated, which is a full snapshot on EC2 - The snapshot will be automatically stored in the AWS S3 bucket created in the first step.
  - Restore this full snapshot to the AWS ES.

- Periodic incremental snapshots
    - Repeat serval incremental snapshot and restore.

- Final snapshot and service switchover
  - Stop services which can modify index data.
  - Create a final incremental snapshot of the EC2.
  - Perform service switchover to the AWS ES.

# Checklist
Here’s a list of what will we do in detail:

## Assess and analyse current data
The ES data is around 30G, setting `number_of_replicas` to 1, then according to the [simplified version of calculation](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/sizing-domains.html):
```
Source Data * (1 + Number of Replicas) * 1.45 = Minimum Storage Requirement
```

So the minimum storage for AWS managed ES is about 90G.

```
curl $es/_cat/indices?v
health status index              pri rep docs.count docs.deleted store.size pri.store.size
green  open   .marvel-2019.10.12   1   1     115081            0      413mb        206.3mb
green  open   site-1.0           200   1     937559          884      1.6gb        848.7mb
green  open   site_v1            200   1     324638           33    335.8mb        167.9mb
green  open   .marvel-2019.10.14   1   1     115009            0    424.5mb        212.2mb
green  open   .marvel-2019.10.15   1   1     114962            0    424.7mb          212mb
green  open   .marvel-2019.10.16   1   1      14290            0     56.3mb         28.3mb
green  open   .marvel-2019.10.13   1   1     115104            0    410.5mb          205mb
green  open   search             200   1   20829799      6167280       24gb           12gb
green  open   .marvel-kibana       1   1          1            0      6.4kb          3.2kb
```

## Create an S3 bucket for current Elasticsearch data (on EC2)
```
e.g. s3://es-backup (us-west-2)
```

## Make ES Cluster snapshot and move it to S3
- check snapshot repository setttings
```
curl localhost:9200/_snapshot?pretty
 
{
  "XXX-snapshot-s3-repo" : {
    "type" : "s3",
    "settings" : {
      "bucket" : "es-backup",
      "region" : "us-west-2"
    }
  }
}
```

- set snapshot repo to S3
```
curl -XPUT localhost:9200/_snapshot/XXX-snapshot-s3-repo?verify=false -d '{
  "type": "s3",
  "settings": {
    "bucket": "es-backup",
    "region": "us-west-2"
  }
}'
```

- check indices
```
curl localhost:9200/_cat/indices?pretty
```

- make snapshots
```
# specified indices
curl -XPUT "localhost:9200/_snapshot/XXX-snapshot-s3-repo/marvel-20191101?wait_for_completion=true" -d '{
  "indices": ".marvel-2019.11.01",
  "ignore_unavailable": "true",
  "include_global_state": false
}'
 
{"snapshot":{"snapshot":"marvel-20191024","indices":[".marvel-2019.10.24"],"state":"SUCCESS","start_time":"2019-10-25T00:20:47.625Z","start_time_in_millis":1571962847625,"end_time":"2019-10-25T00:21:06.058Z","end_time_in_millis":1571962866058,"duration_in_millis":18433,"failures":[],"shards":{"total":1,"failed":0,"successful":1}}}
```
 
- check snapshots
```
# check snapshots
 
curl localhost:9200/_snapshot/XXX-snapshot-s3-repo/_all?pretty
{
  "snapshots" : [ {
    "snapshot" : "marvel-20191016_20191018",
    "indices" : [ "site_v1" ],
    "state" : "SUCCESS",
    "start_time" : "2019-10-18T01:56:21.344Z",
    "start_time_in_millis" : 1571363781344,
    "end_time" : "2019-10-18T01:57:56.012Z",
    "end_time_in_millis" : 1571363876012,
    "duration_in_millis" : 94668,
    "failures" : [ ],
    "shards" : {
      "total" : 200,
      "failed" : 0,
      "successful" : 200
    }
  } ]
}
```
 
## Create and configure AWS Elasticsearch service (version 1.5)

## Migrate the data (Restore cluster from S3 to AWS ES)

- check current iam instance profile role on XXX ES hosts
```
aws sts get-caller-identity
{
    "Account": "XXXXXXXXXXX",
    "UserId": "AROAISLXH37RO7WDQU7H2:i-f9f6cf20",
    "Arn": "arn:aws:sts::XXXXXXXXXXX:assumed-role/elasticsearch-cloud-aws/i-f9f6cf20"
}
```

- set snapshot repository for AWS managed ES

Must sign your snapshot requests, if your access policies specify IAM users or roles.

p.s. If using curl will get the following error message:

> {"Message":"User: anonymous is not authorized to perform: iam:PassRole on resource: arn:aws:iam::XXXXXXXXXXX:role/test-role"}
 
Example code:

```
python register_es_repo.py
 
import boto3
import requests
import json
from requests_aws4auth import AWS4Auth
 
host = 'https://vpc-es-XXX-prod-ji6wgswtt3btfue5pwqwgvudry.us-west-2.es.amazonaws.com/'
region = 'us-west-2'
service = 'es'
credentials = boto3.Session().get_credentials()
awsauth = AWS4Auth(credentials.access_key, credentials.secret_key, region, service, session_token=credentials.token)
 
# Register repository
path = '_snapshot/XXX-snapshot-s3-repo' # the Elasticsearch API endpoint
url = host + path
 
payload = {
  "type": "s3",
  "settings": {
    "bucket": "es-backup",
    "region": "us-west-2",
    "role_arn": "arn:aws:iam::XXXXXXXXXX:role/elasticsearch-cloud-aws"
  }
}
 
headers = {"Content-Type": "application/json"}
 
r = requests.put(url, auth=awsauth, json=payload, headers=headers)
 
print(r.status_code)
print(r.text)
 
 
# Output
200
{"acknowledged":true}
```

- check snapshot repository settings
```
curl -XGET $es/_snapshot?pretty
 
{
  "XXX-snapshot-s3-repo" : {
    "type" : "s3",
    "settings" : {
      "bucket" : "es-backup",
      "role_arn" : "arn:aws:iam::XXXXXXXXXXX:role/aws-elasticsearch-backup",
      "region" : "us-west-2"
    }
  }
}
```

- update search index settings to speed up restore process

```
# https://aws.amazon.com/premiumsupport/knowledge-center/elasticsearch-indexing-performance/
# For huge index, reduce from 30min to 10min together with incremental snapshot:
 
# before migration
curl -XPUT "$es/search/_settings" -d' {
        "number_of_replicas" : 0,
        "refresh_interval" : "-1"
}'
 
# set it back after migration
curl -XPUT "$es/search/_settings" -d' {
        "number_of_replicas" : 1,
        "refresh_interval" : "1s"
}'
```

- restore snapshots from S3 into AWS managed ES
```
# e.g restore from specific snapshot
curl -XPOST "$es/_snapshot/XXX-snapshot-s3-repo/marvel-20191024/_restore"

# e.g. restore just one index, ".marvel-2019.10.24", from "marvel-20191024" snapshot in the "XXX-snapshot-s3-repo" snapshot repository:
curl -XPOST "$es/_snapshot/XXX-snapshot-s3-repo/marvel-20191024/_restore" -d '{"indices": ".marvel-2019.10.24"}' -H 'Content-Type: application/json'
 
# e.g restore all indices except for the .kibana index
curl -XPOST "$es/_snapshot/XXX-snapshot-s3-repo/marvel-20191024/_restore" -d '{"indices": "*,-.kibana"}' -H 'Content-Type: application/json'
```

- Run `es_migrate.sh` script to snapshot / restore

```
time bash -x es_migrate.sh
 
real    10m7.667s
user    0m0.129s
sys 0m0.142s
```

## Switch from self-hosted to AWS managed ES
get ECS task parameter settings and set ES endpoint to AWS managed ES for application

## Testing
- General function testing
- Rollback to old ELB in CNAME if errors
- Data Integration Check

Will generate `doc_count` diff result for old / new ES (take index `.marvel-*` for example):

```
#index  old_doc_count   new_doc_count   diff_rate
 
.marvel-2019.11.08  114929      114929      0.0000%
.marvel-2019.11.09  114930      114930      0.0000%
.marvel-2019.11.10  114904      114904      0.0000%
.marvel-2019.11.11  104327      103532      -0.7620%
search              21025214    21025162    -0.0002%
site-1.0            939656      939657      0.0001%
site_v1             324638      324638      0.0000%
 
real    10m7.303s
user    0m0.156s
sys 0m0.121s
```

## Tidy up resources in Puppet / Terraform
The last step is that clean up old resources in Puppet or Terraform (e.g. EC2 / SG / Route53 etc.)

# Reference
[knowledge-center/elasticsearch-indexing-performance](https://aws.amazon.com/premiumsupport/knowledge-center/elasticsearch-indexing-performance/)

[AWS - sizing domains](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/sizing-domains.html)

[Migrating your self-hosted ElasticSearch to AWS ElasticSearch Service](https://medium.com/krakensystems-blog/migrating-your-self-hosted-elasticsearch-to-aws-elasticsearch-service-a-detailed-guide-ff22efede5a3)