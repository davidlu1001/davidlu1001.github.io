---
title: ElasticSearch Runbook
tags:
  - ElasticSearch
  - SRE
categories:
  - [SRE]
  - [ElasticSearch]
abbrlink: a1a5b7b2
date: 2020-04-16 10:07:57
---

ElasticSearch can be a beast to manage. Knowing the most used endpoints during outages or simple maintenance by heart can be as challenging as it is time consuming. Because of this, this How-To article will layout some handy commands. In theory you could run them from any given node (data, client or master), however I'd recommend running them from a master node.

ssh into any master node (pro-tip: master instances are the ones within the master autoscaling group)

e.g.
```
ssh elastic-master-<instance-id>.<aws-region>.x.y.com
```

## Handy `aliases`
```
alias tail-es='tail -500f /var/log/elasticsearch/<ES_NAME>/<ES_Name>.log'
alias cat-nodes='curl -s -XGET http://localhost:9200/_cat/nodes?v'
alias cat-shards='curl -s -XGET http://localhost:9200/_cat/shards?h=index,shard,prirep,state,unassigned.reason'
alias cat-allocation='curl -s -XGET http://localhost:9200/_cat/allocation?v'
alias cat-indices='curl -s -XGET http://localhost:9200/_cat/indices?v'
alias get-templates='curl -s -XGET http://localhost:9200/_template?pretty'
alias cat-settings='curl -s -XGET http://localhost:9200/_cluster/settings/?pretty'
alias cat-settings-all='curl -s -XGET '\''http://localhost:9200/_cluster/settings?include_defaults=true&pretty'\'' | jq .'
alias cluster-status='curl -s -XGET http://localhost:9200/_cluster/health?pretty'
alias check-status='while true; do sleep 5; cluster-status | grep status; done'
alias disable-allocation='curl -XPUT localhost:9200/_cluster/settings -d '\''{"transient" : \{"cluster.routing.allocation.enable" : "none"}}'\'''
alias enable-allocation='curl -XPUT localhost:9200/_cluster/settings -d '\''{"transient" : \{"cluster.routing.allocation.enable" : "all"}}'\'''
alias cat-shards-unassigned='curl -s -XGET http://localhost:9200/_cat/shards | grep UNASSIGNED | awk '\''{print $1}'\'''
alias del-shards-unassigned='curl -s -XGET http://localhost:9200/_cat/shards | grep UNASSIGNED | awk '\''{print $1}'\'' | xargs -i curl -s -XDELETE "http://localhost:9200/{}"'
alias cat-snapshot='curl -s -XGET localhost:9200/_snapshot/<S3_BUCKET_FOR_SNAPSHOT_REPOSITORY>/_all?pretty | jq .snapshots[-1]'
```

<!-- more -->

## Cheatsheet
https://github.com/sematext/cheatsheets/blob/master/elasticsearch-devops-cheatsheet.md

## Node types
Elasticsearch nodes can take one ore more roles. Here we are using the following node types:

- Master

The master node is responsible for lightweight cluster-wide actions such as creating or deleting an index, tracking which nodes are part of the cluster, and deciding which shards to allocate to which nodes. It is important for cluster health to have a stable master node. Any master-eligible node (all nodes by default) may be elected to become the master node by the master election process.

- Data 

Data nodes hold the shards that contain the documents you have indexed. Data nodes handle data related operations like CRUD, search, and aggregations. These operations are I/O-, memory-, and CPU-intensive. It is important to monitor these resources and to add more data nodes if they are overloaded. 

Can be subdivided into warm and hot nodes if wanting to [Implement a Hot-Warm-Cold Architecture for ES](https://www.elastic.co/blog/implementing-hot-warm-cold-in-elasticsearch-with-index-lifecycle-management)

- Client

Can only route requests, handle the search reduce phase, and distribute bulk indexing. Essentially, coordinating (aka client) only nodes behave as smart load balancers. 

## Cluster health
To check the overall cluster health (green, yellow or red) and keep polling for status just type cluster-status.

```
elastic-master:~$ check-status
  "status" : "green",
  "status" : "green",
  "status" : "green",
```

## Tailing logs
Elasticsearch logs live in `/var/log/elasticsearch/<cluster-name>/<cluster-name>.log`. See the alias `tail-es` above.

## Listing allocations
To list the allocations per host, just type `cat-allocation`.
This will show things like:

- shards per host
- used disk
- available disk
- indice size
- total disk size
- disk usage percent
  - This is an important metric. as a rule of thumb, once disk usage reaches `85%` things starts to go wild. ES will not allocate new shards to nodes once they have more than 85% disk used.

```
elastic-master:~$ cat-allocation
shards disk.indices disk.used disk.avail disk.total disk.percent host          ip            node
   927      434.5gb   472.8gb    265.2gb    738.1gb           64 172.21.5.57   172.21.5.57   elastic-data-096596c27d5ea41b8-production
   928      440.8gb   479.1gb    258.9gb    738.1gb           64 172.21.22.28  172.21.22.28  elastic-data-015b67ccc6632775b-production
   927      496.4gb   534.7gb    203.3gb    738.1gb           72 172.21.21.209 172.21.21.209 elastic-data-02b5fb2c632eacc9f-production
   927      463.8gb     502gb      236gb    738.1gb           68 172.21.5.138  172.21.5.138  elastic-data-07aa318c82d04e61c-production
   927      436.2gb     475gb      263gb    738.1gb           64 172.21.39.147 172.21.39.147 elastic-data-0a7381067cc43e698-production
   928      457.5gb   495.8gb    242.2gb    738.1gb           67 172.21.5.32   172.21.5.32   elastic-data-05a9cd4cea5328486-production
   927      419.1gb   457.3gb    280.7gb    738.1gb           61 172.21.21.160 172.21.21.160 elastic-data-0abf96b0106bcba98-production
   928      447.2gb     485gb      253gb    738.1gb           65 172.21.38.167 172.21.38.167 elastic-data-0b00da1daa6c3e055-production
   927      472.8gb   510.6gb    227.4gb    738.1gb           69 172.21.39.249 172.21.39.249 elastic-data-09512dff9c057e62b-production
```

## Deleting corrupted / lost shards
Sometimes shards can get corrupted or entirely lost. This will put the cluster in a `RED/YELLOW` state. If no other node have the unassigned shard and index, then the only option will be to delete them. To do so, run the alias `del-shards-unassigned` mentioned above or the command below:

```
curl -s -XGET http://localhost:9200/_cat/shards | grep UNASSIGNED | awk {'print $1'} | xargs -i curl -s -XDELETE "http://localhost:9200/{}"
```

## Listing nodes
To list the nodes participating in a cluster, just type `cat-nodes`. The node with a star (`*`) is the actual master within that cluster.

```
elastic-master:~$ cat-nodes
host          ip            heap.percent ram.percent load node.role master name
172.21.38.167 172.21.38.167           67          99 8.84 d         -      elastic-data-0b00da1daa6c3e055-production
172.21.5.57   172.21.5.57             22          99 4.27 d         -      elastic-data-096596c27d5ea41b8-production
172.21.39.60  172.21.39.60            48          91 0.00 -         m      elastic-master-033de9aa1c6f03b8a-production
172.21.22.28  172.21.22.28            74          99 6.41 d         -      elastic-data-015b67ccc6632775b-production
172.21.39.147 172.21.39.147           21          99 5.07 d         -      elastic-data-0a7381067cc43e698-production
172.21.36.186 172.21.36.186           25          98 0.06 -         -      elastic-client-0bdfaa8f29b51ee5c-production
172.21.6.250  172.21.6.250            27          87 0.02 -         -      elastic-client-0579bf60b8746b365-production
172.21.20.96  172.21.20.96             9          84 0.01 -         -      elastic-client-0a25e84d941f3446f-production
172.21.5.138  172.21.5.138            62          99 6.83 d         -      elastic-data-07aa318c82d04e61c-production
172.21.7.188  172.21.7.188            20          93 0.00 -         *      elastic-master-049a20b4645a557eb-production
172.21.21.160 172.21.21.160           17          99 7.49 d         -      elastic-data-0abf96b0106bcba98-production
172.21.5.32   172.21.5.32             66          99 6.33 d         -      elastic-data-05a9cd4cea5328486-production
172.21.22.253 172.21.22.253           38          85 0.00 -         m      elastic-master-0644394056fa488a8-production
172.21.21.209 172.21.21.209           19          99 5.47 d         -      elastic-data-02b5fb2c632eacc9f-production
172.21.39.249 172.21.39.249           61          99 4.11 d         -      elastic-data-09512dff9c057e62b-production
```

## Stucked nodes
We have come across an issue where one or more nodes get stucked, preventing the master from reallocating shards properly. This has happened multiple times. To find out which nodes are stucked you can either search for the exception `ReceiveTimeoutTransportException` from logs or search in Kibana if applicable.

## Decomissioning Node
Node decomission is something you do when you "empty" a node, a.k.a, force shard reallocation. To do that just use one of the commands bellow, depending whether you want to exclude a node by `ip` or by `name`. Using `name` is the preferred way.

Both attributes, ip or name, can be retrieved from the `cat-nodes` alias mentioned above. You usually want to do this for one host at a time.

```
# exclude by IP
curl -XPUT localhost:9200/_cluster/settings -d '{
  "transient" :{
      "cluster.routing.allocation.exclude._ip" : "172.19.22.9"
   }
}'

# exclude by NAME
curl -XPUT localhost:9200/_cluster/settings -d '{
  "transient" :{
      "cluster.routing.allocation.exclude._name" : "elastic-data-05a4f739b4b5fc4ab"
   }
}'
```

More info could be found: [ElasticSearch - cluster-level shard allocation-filtering](https://www.elastic.co/guide/en/elasticsearch/reference/current/allocation-filtering.html)

## Rolling restart

1. Disable shard allocation

```
curl -XPUT localhost:9200/_cluster/settings -d '{
"transient" : {
"cluster.routing.allocation.enable" : "none"
}
}'
```

2. Stop node (check the alias command on the host to stop a node)

3. Perform maintenance / upgrade

4. Restart the node, and confirm that it joined the cluster.

5. Re-enable shard allocation as follows:

```
curl -XPUT localhost:9200/_cluster/settings -d '{
"transient" : {
"cluster.routing.allocation.enable" : "all"
}
}'
```

The above steps can also be done from `ansible playbook` as follows:

```
---
- name: Perform a rolling restart of an Elasticsearch cluster
  gather_facts: no
  hosts: elastic-client-es7:elastic-master-es7:elastic-data-warm-es7:elastic-data-hot-es7
  serial: 1
  become: yes

  vars_prompt:
    - name: "service"
      prompt: "Enter the cluster name (eg. A|B|C)"
      private: no
      default: "unknown"

  vars:
    es_disable_allocation: '{"transient":{"cluster.routing.allocation.enable": "none"}}'
    es_enable_allocation: '{"transient":{"cluster.routing.allocation.enable": "all"}}'
    es_http_port: 9200
    es_transport_port: 9300

  tasks:
    - name: Ensure Elasticsearch is running on node
      service: name=elasticsearch-{{ service }} enabled=yes state=started
      register: response

    - name: Wait for node to come back up if it was stopped
      wait_for: port={{ es_transport_port }} delay=45
      when: response.changed == true

    # the ansible the uri action needs httplib2
    - name: ensure python-httplib2 is installed
      apt: name=python-httplib2 state=present

    - name: Check current version
      uri: url=http://localhost:{{ es_http_port }} method=GET
      register: version_found
      retries: 10
      delay: 10

    - name: Display current Elasticsearch version
      debug: var=version_found.json.version.number

    # enable first - in case the shards were disabled before for some reason.
    # it's important because the next task will wait for the cluster to become green.
    # If the shard allocation is disabled the cluster will stay yellow and the tasks will hang until it times out.
    - name: Enable shard allocation for the {{ service }} cluster
      uri:
        url: http://localhost:{{ es_http_port }}/_cluster/settings
        method: PUT
        body_format: json
        body: "{{ es_enable_allocation }}"

    - name: Wait for cluster health to return to green
      uri:
        url: http://localhost:{{ es_http_port }}/_cluster/health
        method: GET
      register: response
      until: "response.json.status == 'green'"
      retries: 500
      delay: 15

    - name: Disable shard allocation for the {{ service }} cluster
      uri:
        url: http://localhost:{{ es_http_port }}/_cluster/settings
        method: PUT
        body_format: json
        body: "{{ es_disable_allocation }}"

    - name: Restart elasticsearch-{{ service }} on node
      service: name=elasticsearch-{{ service }} enabled=yes state=restarted

    - name: Wait for node to come back up
      wait_for: port={{ es_transport_port }} delay=35

    - name: Wait for http to come back up
      wait_for: port={{ es_http_port }} delay=5

    - name: Wait for cluster health to return to yellow or green
      uri: url=http://localhost:{{ es_http_port }}/_cluster/health method=GET
      register: response
      until: "response.json.status == 'yellow' or response.json.status == 'green'"
      retries: 500
      delay: 15

    - name: Re-enable shard allocation for the {{ service }} cluster
      uri:
        url: http://localhost:{{ es_http_port }}/_cluster/settings
        method: PUT
        body_format: json
        body: "{{ es_enable_allocation }}"
      register: response
      until: "response.json.acknowledged == true"
      retries: 10
      delay: 15

    - name: Wait for the node to recover
      uri:
        url: http://localhost:{{ es_http_port }}/_cat/health
        method: GET
        return_content: yes
      register: response
      until: "'green' in response.content"
      retries: 500
      delay: 30
```

Then we can run the ansible playbook to perform the rolling restart for Elasticsearch

```
ansible-playbook \
	-i inventory/aws/production/a-us-west-2 \
	--ask-become-pass \
	playbooks/restart_elasticsearch_es7.yml
```