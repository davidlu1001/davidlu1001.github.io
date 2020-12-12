---
title: ElasticSearch backup with AWS S3
tags:
  - ElasticSearch
  - Backup
  - DR
  - S3
  - curator
  - SRE
categories:
  - - SRE
  - - ElasticSearch
  - - Curator
abbrlink: 11c588c4
date: 2020-12-12 00:08:50
---

# TL;DR

- ElasticSearch Backup (snapshot / restore) on AWS S3

- Steps / Configrations for ES snapshot / restore

- Use elastic `curator` to manage snapshots (create / remove)

- Docker image for ES Curator to manage Elasticsearch snapshots - [davidlu1001/docker-curator](https://github.com/davidlu1001/docker-curator)

# Overview

The purpose of this blog is to investigate the possible solutions to backup and restore ES indices. So that in the event of a failure, the cluster data can be quickly restored and minimized the business impact.

## Backup content
Generally the backup content would include:

- Cluster data

- State configuration: includes cluster / index / shard settings

```
# index settings
curl -X GET "localhost:9200/_all/_settings?pretty"

# cluster settings
curl -s -XGET "http://localhost:9200/_cluster/settings?pretty&flat_settings&filter_path=persistent"
{
  "persistent" : {
    "cluster.routing.allocation.cluster_concurrent_rebalance" : "12",
    "cluster.routing.allocation.node_concurrent_recoveries" : "6",
    "indices.recovery.max_bytes_per_sec" : "320mb"
  }
}
```

P.S.

- Transient settings are not considered for backup

- Also can capture these cluster settings in a data backup snapshot by specifying the `include_global_state: true` (default) parameter for the snapshot API.

## Snapshot Repository
In order to enable backup for ES cluster data, a snapshot repository must be registered before performing snapshot and restore operations.

Snapshots can be stored in:

- Local shared filesystem: NFS

- Remote repositories: backed by the cloud (AWS / Azure / GCP) or by distributed file systems (HDFS) with [repository plugins](https://www.elastic.co/guide/en/elasticsearch/plugins/7.10/repository.html)

Considering that if use shared filesystem NFS, additional setup / configuration is required, and the cluster needs rolling restart to take effect. As all nodes must have access to the shared storage to be able to store the snapshot data. Therefore, from the perspective of complexity and operational safety, will not consider it (even the cluster only uses a little of the disk space).

And luckily we’ve got `repository-s3` plugin installed on all nodes in the ES cluster, so rolling restart is not needed if using S3 as snapshot repository.

```
$ ES_PATH_CONF=/etc/elasticsearch /usr/share/elasticsearch/bin/elasticsearch-plugin list

discovery-ec2
repository-s3
```

So would prefer to use AWS S3 as a remote snapshot repository with existing S3 repository plugin.

Backup retention strategy
Based on backup granularity and retention time, we can choose the following strategies based on different scenario:

- Takes daily snapshots (during the hour we specify) and retains up to 14 of them for 30 days.

- Takes hourly snapshots and retains up to 336 of them for 14 days.

Can carefully start with daily backup (the initial backup may take relatively longer time) and keep monitoring, and then could try hourly backup if it’s feasible, in order to provide more granular recovery points.

> Incremental snapshot mechanism
>
> The snapshot functionality will simply remove the data that was only referenced by the deleted snapshot but will leave the data that is still required for other snapshots in the repository.
>
> So it would be safe to delete the first initial snapshot without affecting the latter snapshots.
>
> Ref: https://www.elastic.co/blog/found-elasticsearch-snapshot-and-restore

# Backup cluster’s data
Will take snapshot per index, instead of backup the whole cluster, which can potentially bring the flexibility for the restore step.

The snapshots are taken incrementally. This enables us to take frequent snapshots with minimal overhead (the initial backup may take relatively longer time, based on the amount of data). The more frequently we take snapshots, the less time it will take to complete the backup.

The snapshot process is executed in non-blocking fashion. All indexing and searching operations can continue to run against the index that is being snapshotted.

## Snapshot Prerequisites

- Check repository-s3 plugin is installed

```
$ ES_PATH_CONF=/etc/elasticsearch /usr/share/elasticsearch/bin/elasticsearch-plugin list

discovery-ec2
repository-s3
```

- Create an S3 bucket to store the snapshot (e.g. `es-backup`)

- Check current iam instance profile role on ES cluster

- Update existing IAM role and setup proper S3 Permissions

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListAllMyBuckets",
                "s3:GetBucketLocation"
            ],
            "Resource": "arn:aws:s3:::*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:ListBucketMultipartUploads",
                "s3:ListBucketVersions"
            ],
            "Resource": [
                "arn:aws:s3:::es-backup"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectAcl",
                "s3:PutObject",
                "s3:PutObjectAcl",
                "s3:DeleteObject",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::es-backup/*"
            ]
        }
    ]
}
```

- Need to update the role permission for both ES master / data node

> Note:
>
> Otherwise will get the repository_verification_exception error when trying to register the snapshot repository

```
    "root_cause" : [
      {
        "type" : "repository_verification_exception",
        "reason" : "[snapshot-s3-repo] [[zMHw44eLS7q0ItSfYmh2Ug, 'RemoteTransportException[[elastic-data-warm-0db25600a6f728fe9][172.17.24.56:9300][internal:admin/repository/verify]]; nested: BlobStoreException[Failed to check if blob [master.dat] exists]; nested: NotSerializableExceptionWrapper[amazon_s3_exception: Forbidden (Service: Amazon S3; Status Code: 403; Error Code: 403 Forbidden; Request ID: 425B663C880BBD13; S3 Extended Request ID: huXmNMJSK+Lh4vW5LIqJsYUfu9s3ZhPa2knSeblOVBbNnoffvYaZXhMz4sgfT8ZGTAAC+3yjUNc=)];']
        ...
        ]"
      }
```

- Register snapshot repository per index with base_path (prefix) in S3 bucket for ES cluster (one-time operation)

```
# use `base_path` under S3 for `service-A`

curl -X PUT "localhost:9200/_snapshot/snapshot-s3-repo?pretty" -H 'Content-Type: application/json' -d'
{
  "type": "s3",
  "settings": {
    "bucket": "es-backup",
    "region": "us-west-2",
    "base_path": "service-A",
    "max_snapshot_bytes_per_sec": "500mb",
    "max_restore_bytes_per_sec": "1gb"
  }
}
'
```

- Register snapshot repository for each index (with base_path: same S3 bucket, but different directory)

```
# e.g. for .elastichq

curl -X PUT "localhost:9200/_snapshot/snapshot-s3-elastichq?pretty" -H 'Content-Type: application/json' -d'
{
  "type": "s3",
  "settings": {
    "bucket": "es-backup",
    "base_path": "elastichq",
    "region": "us-west-2",
    "max_snapshot_bytes_per_sec": "500mb",
    "max_restore_bytes_per_sec": "1gb"
  }
}
'
```

- Check or Verify snapshot repository
```
curl localhost:9200/_snapshot?pretty
```

# Backup operation
The snapshots lifecycle management (SLM) feature is introduced and natively supported in ES version `7.5.0`, so for lower version need to self-manage the snapshots / retention policy.

Can either use existing tool curator (with cronjob) on EC2 instance (e.g. gateway / master node), or can consider using ECS cronjob (scheduled task) to schedule the backup to avoid single point of failure.

For curator version compatibility, should be safe to choose the latest version `5.8.3` to support ES `6.X`.

Backup operation with ES snapshot API:

```
# e.g. backup specified indice

curl -X PUT "localhost:9200/_snapshot/snapshot-s3-repo/elastichq-20201125?wait_for_completion=false&pretty" -H 'Content-Type: application/json' -d'
{
  "indices": ".elastichq",
  "ignore_unavailable": true,
  "include_global_state": false
}
'

# check snapshots

curl localhost:9200/_snapshot/snapshot-s3-repo/_all?pretty

# delete snapshot

curl -X DELETE "localhost:9200/_snapshot/snapshot-s3-repo/elastichq-20201125?pretty"
```

> Useful Config for Snapshot / Restore Process
>
> 
> `max_restore_bytes_per_sec`
> 
> Throttles per node restore rate. Defaults to unlimited. Note that restores are also throttled through [recovery settings](https://www.elastic.co/guide/en/elasticsearch/reference/7.10/recovery.html).
> 
> `max_snapshot_bytes_per_sec`
> 
> Throttles per node snapshot rate. Defaults to `40mb` per second.
> 
> `chunk_size`
> 
> Big files can be broken down into chunks during snapshotting if needed. Specify the chunk size as a value and unit, for example: 1GB, 10MB, 5KB, 500B. Defaults to `1GB`.
> 
> `buffer_size`
> 
> Minimum threshold below which the chunk is uploaded using a single request. Beyond this threshold, the S3 repository will use the [AWS Multipart Upload API](https://docs.aws.amazon.com/AmazonS3/latest/dev/uploadobjusingmpu.html) to split the chunk into several parts, each of buffer_size length, and to upload each part in its own request. Should between 5mb to 5gb. Defaults to the minimum between 100mb and 5% of the heap size.

## Restore cluster’s data

### Restore Prerequisites

- Confirm the target index has same number of shards as the index in the snapshot

- Close the target index need to be restored

> Reference
> 
> By default, the cluster state is not restored. To include the global cluster state, need to set `include_global_state` to `true` in the restore request body (if include the cluster state in the previous backup operation), but usually we don’t want to backup the cluster state.
>
> An existing index can be only restored if it’s closed and has the same number of shards as the index in the snapshot. 
>
> The restore operation automatically opens restored indices if they were closed, and creates new indices if they didn’t exist in the cluster.

### Restore operation

In order to restore the index from a snapshot:

- Update index settings to speed up restore process:

```
# before restore
curl -XPUT "localhost:9200/{index_name}/_settings" -d' {
        "number_of_replicas" : 0,
        "refresh_interval" : "-1"
}'
```

- Restore snapshot from S3 (Set the index settings back at the same time)

```
# e.g restore from specific snapshot service-A

curl -XPOST "localhost:9200/_snapshot/snapshot-s3-repo/service-A-2020112416/_restore?pretty" -H 'Content-Type: application/json' -d'
{
  "indices": "service-A",
  "ignore_unavailable": true,
  "include_global_state": false,
  "include_aliases": false,
  "index_settings": {
    "index.number_of_replicas": 2,
    "index.refresh_interval": "1s"
  }
}
'
```

P.S.

- Use the `index_settings` parameter to override index settings during the restore process

- Set `include_aliases` to `false` to prevent aliases from being restored together with associated indices


# Curator Usage

## use `curator_cli`

Examples:

Here the variable `"${DRY_RUN}"` can be either `"--dry-run"` (for dry-run mode) or `""` (empty, means without dry-run)

- create snapshot

```
# create snapshot for index(es) in the same repo

/usr/local/bin/curator_cli \
	${DRY_RUN} \
	--host "${ELASTICSEARCH_HOST}" \
	--port 9200 \
	snapshot \
	--repository "${REPO_NAME}" \
	--name "${INDEX_PREFIX}-${curr_date}" \
	--wait_for_completion --skip_repo_fs_check \
	--filter_list "{\"filtertype\":\"pattern\",\"kind\":\"prefix\",\"value\":\"${INDEX_PREFIX}\"}"
```

- remove snapshot
```
/usr/local/bin/curator_cli \
	${DRY_RUN} \
	--host "${ELASTICSEARCH_HOST}" \
	--port 9200 \
	delete_snapshots \
	--repository "${REPO_NAME}" \
	--ignore_empty_list \
	--filter_list "[{\"filtertype\":\"age\",\"source\":\"creation_date\",\"direction\":\"older\",\"unit\":\"${UNIT}\",\"unit_count\":\"${UNIT_COUNT}\"},{\"filtertype\":\"pattern\",\"kind\":\"prefix\",\"value\":\"${INDEX_PREFIX}\"}]"
```

- restore snapshot

```
# close first

/usr/local/bin/curator_cli \
	${DRY_RUN} \
	--host "${ELASTICSEARCH_HOST}" \
	--port 9200 \
	close \
	--ignore_empty_list \
	--filter_list "{\"filtertype\":\"pattern\",\"kind\":\"prefix\",\"value\":\"${INDEX_PREFIX}\"}"

# restore

/usr/local/bin/curator_cli \
	${DRY_RUN} \
	--host "${ELASTICSEARCH_HOST}" \
	--port 9200 \
	restore \
	--repository "${REPO_NAME}" \
	--wait_for_completion --skip_repo_fs_check --ignore_empty_list \
	--filter_list "[{\"filtertype\":\"state\"},{\"filtertype\":\"pattern\",\"kind\":\"prefix\",\"value\":\"${INDEX_PREFIX}\"}]"
```

## use `curator`

Examples:

Here the variable `"${DRY_RUN}"` can be either `"--dry-run"` (for dry-run mode) or `""` (empty, means without dry-run)

```
/usr/local/bin/curator --config /etc/curator/config.yml "${DRY_RUN}" /etc/curator/actions.yml
```

`actions.yml` for `snapshot`:

```
# snapshot

actions:
  1:
    action: snapshot
    description: >-
      Snapshot company-index-A-* prefixed indices with the default snapshot name pattern of
      'company-index-A-%Y%m%d%H%M'.  Wait for the snapshot to complete.  Skip
      the repository filesystem access check.  Use the other options to create
      the snapshot.
    options:
      repository: snapshot-s3-index-A
      # Leaving name blank will result in the default 'curator-%Y%m%d%H%M'
      name: company-index-A-%Y%m%d%H%M
      ignore_unavailable: False
      include_global_state: False
      partial: False
      wait_for_completion: True
      skip_repo_fs_check: True
      disable_action: False
    filters:
      - filtertype: pattern
        kind: prefix
        value: company-index-A
  2:
    action: delete_snapshots
    description: >-
      Delete snapshots from the selected repository older than 14 days
      (based on creation_date), for 'company-index-A-*' prefixed snapshots.
    options:
      repository: snapshot-s3-index-A
      # Leaving name blank will result in the default 'curator-%Y%m%d%H%M'
      disable_action: False
      ignore_empty_list: True
    filters:
      - filtertype: pattern
        kind: prefix
        value: company-index-A
      - filtertype: age
        source: creation_date
        direction: older
        unit: days
        unit_count: 14
```

`actions.yml` for `restore`:

```
actions:
  1:
    action: close
    description: "Close selected indices company-index-A before restoring snapshot"
    options:
      continue_if_exception: True
      ignore_empty_list: True
    filters:
      - filtertype: pattern
        kind: prefix
        value: company-index-A
  2:
    action: restore
    description: >-
      Restore all indices in the most recent company-index-A-* snapshot with state
      SUCCESS.  Wait for the restore to complete before continuing.  Skip
      the repository filesystem access check.  Use the other options to define
      the index/shard settings for the restore.
    options:
      repository: snapshot-s3-index-A
      # Leaving name blank will result in restoring the most recent snapshot by age
      name:
      # Leaving indices blank will result in restoring all indices in the snapshot
      indices:
      include_aliases: False
      ignore_unavailable: False
      include_global_state: False
      partial: False
      extra_settings:
        index_settings:
          number_of_replicas: 2
      wait_for_completion: True
      skip_repo_fs_check: True
      disable_action: False
    filters:
      - filtertype: pattern
        kind: prefix
        value: company-index-A
        exclude:
      - filtertype: state
        state: SUCCESS
        exclude:
```

## docker-curator

The Docker image for ES Curator (to manage ES `snapshots`) could be found here - [davidlu1001/docker-curator](https://github.com/davidlu1001/docker-curator)

This image keeps up to date with curator releases `5.8.3`. It is also based on minimal alpine image.

### Features
- Upgrade curator to version `5.8.3`
- Add support for snapshot / restore (use `curator_cli` for single index scenario)
- Add support for snapshot / restore `ALL` indexes for ES using `curator` with actions rules. This would be useful when:
    - too many indexes (can not match with `prefix / regex` pattern) for `curator_cli`
    - if use different snapshot repository per index
    - for accident recovery scenario to restore ALL indexes
- Add `DRY_RUN` mode
- Rewrite Dockerfile and use `alpine` to reduce image size (with `python3`)

### Usage
Image `entrypoint` is set to customized script, need to pass paremeters to `CMD`, can also support override `ENV`

Default ENV value:
```
TYPE=snapshot
INDEX_PREFIX=.kibana
REPO_NAME=snapshot-repo
DRY_RUN=True
```

e.g.
```
# Snapshot single index with DRY_RUN mode, and delete snapshots 14 days ago

docker-compose run --rm es-curator snapshot .monitoring-es-7-2020.12.04 snapshot-repo True


# Restore single index (with latest snapshot) without DRY_RUN mode

docker-compose run --rm es-curator restore .monitoring-es-7-2020.12.04 snapshot-repo False


# Snapshot ALL indexes for ES without DRY_RUN mode, and delete snapshots 14 days ago

docker-compose run --rm es-curator snapshot ALL snapshot-repo False


# Restore ALL indexes for ES (with latest snapshot) without DRY_RUN mode

docker-compose run --rm es-curator restore ALL snapshot-repo False
```

Pass `ENV`:

```
- ELASTICSEARCH_HOST: default is `elasticsearch`

- UNIT: default is days, support `seconds | minutes | hours | days | weeks | months | years`

- UNIT_COUNT: default is 14
```

e.g.
```
# Snapshot single index without DRY_RUN mode, and delete snapshots older than 1 minutes ago

UNIT=minutes UNIT_COUNT=1 docker-compose run --rm es-curator snapshot .monitoring-es-7-2020.12.04 snapshot-repo False
```

# References

- [Elastic - Backup Cluster](https://www.elastic.co/guide/en/elasticsearch/reference/current/backup-cluster.html)

- [S3 repository plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/7.10/repository-s3.html)

- [Repository settings](https://www.elastic.co/guide/en/elasticsearch/plugins/current/repository-s3-repository.html)

- [ES - Snapshot and Restore (incremental snapshot mechanism)](https://www.elastic.co/blog/found-elasticsearch-snapshot-and-restore)

- [Elastic Curator](https://www.elastic.co/guide/en/elasticsearch/client/curator/5.8/index.html)