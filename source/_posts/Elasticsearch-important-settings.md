---
title: Elasticsearch important settings
tags:
  - Elasticsearch
categories: SRE
abbrlink: aa4fabe3
date: 2020-04-08 23:36:06
---

Document important settings we shouldn't be missing in elasticsearch clusters.

# Overview

## Cluster settings for ES

Could use `Kibana` Dev Tool to update cluster settings

```
{
  "persistent" : {
    "cluster" : {
      "routing" : {
        "allocation" : {
          "cluster_concurrent_rebalance" : "12",
          "node_concurrent_recoveries" : "6",
          "disk" : {
            "watermark" : {
              "high" : "85%"
            }
          },
          "enable" : "all"
        }
      }
    },
    "indices" : {
      "recovery" : {
        "max_bytes_per_sec" : "640mb"
      }
    }
  },
  "transient" : {
    "cluster" : {
      "routing" : {
        "allocation" : {
          "cluster_concurrent_rebalance" : "12",
          "include" : {
            "_name" : "elastic-data-*",
            "_ip" : ""
          },
          "node_concurrent_recoveries" : "6",
          "balance" : {
            "threshold" : "1.0f"
          },
          "enable" : "all"
        }
      }
    },
    "indices" : {
      "recovery" : {
        "max_bytes_per_sec" : "640mb"
      }
    }
  }
}
```

## indices.recovery.max_bytes_per_sec

Defaults to `40mb`. Increasing to `320mb` increases throughput

```
curl -X PUT "localhost:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
{
    "persistent" : {
        "indices.recovery.max_bytes_per_sec" : "320mb"
    }
}'
```

# cluster.routing.allocation.cluster_concurrent_rebalance

By default ES will only move `2` shards at any given time.

To increase recovery speeds and shard rebalancing we should bump this up.

```
curl -XPUT localhost:9200/_cluster/settings?pretty -H 'Content-Type: application/json' -d '{
  "persistent" : {
    "cluster.routing.allocation.cluster_concurrent_rebalance": 12
  }
}'
```