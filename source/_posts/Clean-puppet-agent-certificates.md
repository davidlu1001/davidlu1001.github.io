---
title: Clean puppet agent certificates
tags:
  - DevOps
  - Puppet
categories: 
  - [DevOps]
  - [Puppet]
abbrlink: dda5d6cb
date: 2018-03-08 23:42:16
---

This document outlines the steps to clean or regenerate puppet agent certificates in a traditional master/client setup.

First thing is to ssh into the agent 

Then, delete all `*.pem` files in `/var/lib/puppet/ssl` associated to your instance.
e.g they should be in the form of <HOSTNAME>.pem

```
root@gateway:/var/lib/puppet/ssl# hostname -f
gateway.ap-southeast-2.aws.ci.xxxxxx.com

root@gateway:/var/lib/puppet/ssl# find . -type f | grep "gateway.ap-southeast-2.aws.ci.xxxxxx.com"
./public_keys/gateway.ap-southeast-2.aws.ci.xxxxxx.com.pem
./private_keys/gateway.ap-southeast-2.aws.ci.xxxxxx.com.pem
./certificate_requests/gateway.ap-southeast-2.aws.ci.xxxxxx.com.pem
./certs/gateway.ap-southeast-2.aws.ci.xxxxxx.com.pem

# to delete them
find . -type f | grep "gateway.ap-southeast-2.aws.ci.xxxxxx.com" | xargs rm -rf
```

Next step is to ssh into the puppet master and do the same

<!-- more -->

```
root@puppet-master:/var/lib/puppet/ssl# find . -type f | grep "gateway"
./ca/signed/gateway.ap-southeast-2.aws.ci.xxxxxx.com.pem
```

Ideally there should be only one file hanging in there.

Then back to the agent and try running a --noop puppet run to force a new certificate request. After this, go back to the puppet master and check for any pending cert waiting for approval.

```
root@puppet-master:/var/log/puppet# puppet cert list | grep "gateway"
  "gateway.ap-southeast-2.aws.ci.xxxxxx.com"                    (SHA256) E8:D2:83:89:34:A4:AD:14:EA:83:73:8A:B9:E6:98:D9:6C:4F:47:C6:07:5D:D6:D0:9A:F1:32:C8:33:74:98:D0
Cool! Now you just need to sign the certificate.

root@puppet-master:/var/log/puppet# puppet cert sign gateway.ap-southeast-2.aws.ci.xxxxxx.com
Notice: Signed certificate request for gateway.ap-southeast-2.aws.ci.xxxxxx.com
Notice: Removing file Puppet::SSL::CertificateRequest gateway.ap-southeast-2.aws.ci.xxxxxx.com at '/var/lib/puppet/ssl/ca/requests/gateway.ap-southeast-2.aws.ci.xxxxxx.com.pem'
```

Lastly, go back to the agent host and try running a couple of puppet runs.

```
sudo puppet agent --enable && sudo puppet agent -tv --noop; sudo puppet agent --disable
```