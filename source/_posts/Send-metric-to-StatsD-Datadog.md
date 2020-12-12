---
title: Send metric to StatsD (Datadog)
tags:
  - StatsD
  - Datadog
  - Monitoring
  - SRE
categories:
  - - SRE
  - - Monitoring
abbrlink: 5c70d9d6
date: 2020-04-22 15:10:41
---

We can simply send custom metric to Datadog without any of the [DogStatsD client libraries](https://docs.datadoghq.com/developers/libraries/#api-and-dogstatsd-client-libraries))

Generally DogStatsD creates a message that contains information about your metric, event, or service check and sends it to a locally installed Agent as a collector. The destination IP address is `127.0.0.1` and the collector port over `UDP` is `8125`.

Based on the [official doc](https://docs.datadoghq.com/developers/dogstatsd/datagram_shell/?tab=metrics#sending-metrics) of Datadog, here's the raw datagram format for metrics, events, and service checks that DogStatsD accepts:

```
<METRIC_NAME>:<VALUE>|<TYPE>|@<SAMPLE_RATE>|#<TAG_KEY_1>:<TAG_VALUE_1>,<TAG_2>
```

So could use `nc` and `socat` on indivial host:

e.g.
```
# use nc

echo -n "kafka.partition_size:123|g|#topic_name:test,partition_number:1,broker_id:28041,hostname:kafka-12345" | nc -w 1 -cu localhost 8125
```

<!-- more -->

To speed up the send period, can reduce the `-w timeout` to `0` for `nc` command, but sometimes found that `nc -w 0` hung, so started to pipe into `socat` as an alternative:

```
# use socat

echo "${METRICS_NAME}:${PARTITION_SIZE}|g|#topic_name:${TOPIC_NAME},partition_number:${PARTITION_NUMBER},broker_id:${BROKER_ID},hostname:${HOSTNAME}" | socat -t 0 STDIN UDP:localhost:8125
```

# Code example


```
# function to send metric with puppet tags

function send_dd_metric() {
    METRICS_NAME="$1"
    value="$2"

    HOSTNAME=$(hostname -f)

    # Pool
    if [[ -f /etc/pool ]]; then
        PUPPET_POOL=$(cat /etc/pool)
    else
        PUPPET_POOL='unknown'
    fi

    # Role
    if [[ -f /etc/role ]]; then
        PUPPET_ROLE=$(cat /etc/role)
    else
        PUPPET_ROLE='unknown'
    fi

    # Datacenter
    if [[ -f /etc/dc ]]; then
        PUPPET_DC=$(cat /etc/dc)
    else
        PUPPET_DC='unknown'
    fi

    AMI_ID=$(curl --connect-timeout 2 --max-time 4 --silent http://169.254.169.254/latest/meta-data/ami-id)

    echo "${METRICS_NAME}:${value}|g|host:${HOSTNAME},puppet_pool:${PUPPET_POOL},puppet_role:${PUPPET_ROLE},puppet_dc:${PUPPET_DC},ami_id:${AMI_ID}" | socat -t 0 STDIN UDP:localhost:8125
}

# send metric to datadog 
send_dd_metric "test.service" 1

# remember also to send metric when no data
send_dd_metric "test.service" 0
```

# Check from Datadog

When metric reporting successfully, we can open Datadog console:
![](https://raw.githubusercontent.com/davidlu1001/davidlu1001.github.io/hexo/uPic/aNCKGQ.png)

and then search for the metric:
![](https://raw.githubusercontent.com/davidlu1001/davidlu1001.github.io/hexo/uPic/rBrQpy.png)

After that we can also create Dashboard and Monitors (with Datadog API or existing Github repo [barkdog](https://github.com/codenize-tools/barkdog)) when necessary.

# Reference

- [Datadog - sending-metrics](https://docs.datadoghq.com/developers/dogstatsd/datagram_shell/#sending-metrics)

- https://gist.github.com/nstielau/966835
