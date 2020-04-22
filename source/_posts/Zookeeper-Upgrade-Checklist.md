---
title: Zookeeper Upgrade Checklist
tags:
  - Zookeeper
  - SRE
categories:
  - [SRE]
  - [ZooKeeper]
abbrlink: 712b4da2
date: 2019-11-22 20:35:46
---

Just making some notes for the upgrade steps for Zookeeper as a memo.

# Prerequisites

- Terraform work complete to create Zookeeper Auto Scaling Group (ASG)

## Autoscaling
Full autoscaling Zookeeper (actually adding new nodes to the Zookeeper ensemble automatically) is a bit tricky currently as a full rolling restart of the Zookeeper cluster would have to be orchestrated without ever losing quorum. Zookeeper 3.5 introduced [dynamic reconfiguration](https://zookeeper.apache.org/doc/r3.5.5/zookeeperReconfig.html) which would most likely make this significantly easier. There are some other options for [autoscaling Zookeeper 3.4](https://www.credera.com/blog/technology-solutions/how-to-automate-zookeeper-in-aws/) but we'd need to weigh up whether that additional complexity is really worth it.

## Puppet 
Using the puppet module: [deric/puppet-zookeeper](https://github.com/deric/puppet-zookeeper)

### Updating Hiera / Puppet
When a new node starts within the ASG Puppet will run but Zookeeper will not be started automatically. Hiera will need to be updated and Puppet run again for Zookeeper to start.

"zookeeper::servers" - Needs to be set to a hash of the complete list of servers in the Zookeeper ensemble. This is typically set at the "dc" level in the hierarchy, e.g.

```
puppet/hieradata-masterless/dc

cat test-us-west-2.yaml
zookeeper::servers:
  100: zookeeper-0cdf08e0d09dd3530.oregon.test.xyz.org
  101: zookeeper-01ff482cf77636240.oregon.test.xyz.org
  102: zookeeper-0749d84c7dfd4d798.oregon.test.xyz.org
```

"zookeeper::id" - Needs to be set to the unique ID of the node within the Zookeeper cluster. This is set at the node level (as it's always specific to a particular node) and needs to match with one (and only one) entry in "zookeeper::servers". The corresponding node configuration for the above "zookeeper::servers" configuration would be

```
puppet/hieradata-masterless/nodes

cat zookeeper-0cdf08e0d09dd3530.yaml
zookeeper::id: 100
 
cat zookeeper-01ff482cf77636240.yaml
zookeeper::id: 101
 
cat zookeeper-0749d84c7dfd4d798.yaml
zookeeper::id: 102
```

# Migration steps

- Adding additional nodes to expand the existing ensemble. Must be repeated once for each new node
    - Start Zookeeper on one of the replacement nodes. The node should join the ensemble as a follower.
    - Execute a Puppet run on each of the existing nodes one at a time. Restart the nodes in order with the lowest ZKID first but restart the leader last.
    - Ensure the nodes are all in sync
- Once the ensemble has been expanded and is in sync the original Trusty nodes can be removed. Starting with an ensemble of six nodes:
    - Stop Zookeeper on one of the original nodes
    - Update the list of servers on the remaining nodes to remove the server that has been stopped.
    - Rolling restart of the cluster with the nodes in order from lowest to highest ZKID. The leader should be restarted last. At this point the ensemble should consist of five nodes.
    - Stop Zookeeper on the final two original Trusty nodes. The three replacement nodes left on line can still form a quorum and should continue to work as expected.
    - Update the list of servers on the remaining nodes to remove the servers that have been stopped.
    - Rolling restart of the cluster with the nodes in order from lowest to highest ZKID. The leader should be restarted last. At this point the ensemble should consist of three nodes.

# Tidy up
once we're certain that we're never going to revert to the original nodes

- Ensure replacement nodes are being monitored by Datadog
- Ensure replacement nodes are sending logs to the Elastic lifesupport cluster
- Ensure bash aliases are updated
- Terraform work to decommission original nodes