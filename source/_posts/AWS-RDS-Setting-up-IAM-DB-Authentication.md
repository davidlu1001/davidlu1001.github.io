---
title: AWS RDS - Setting up IAM DB Authentication
tags:
  - AWS
  - DB
  - RDS
  - SRE
categories: 
  - [SRE]
  - [AWS]
  - [DB]
abbrlink: ee94cd4e
date: 2020-03-13 23:20:12
---

Firstly we need to create a RDS database account (user) within the database and associate it to 1-N IAM authentication. Below are just policies samples that the module will create behind the scenes. An example of this can be found here.

sample policy granting access to a database instance
```
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Action": [
             "rds-db:connect"
         ],
         "Resource": [
             "arn:aws:rds-db:us-west-2:123456789012:dbuser:db-12ABC34DEFG5HIJ6KLMNOP78QR/david_lu"
         ]
      }
   ]
}
```

sample policy granting access to a cluster
```
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Action": [
             "rds-db:connect"
         ],
         "Resource": [
             "arn:aws:rds-db:us-west-2:123456789012:dbuser:cluster-CO4FHMOYDKJ7CVBEJS2UWDQX7I/david_lu"
         ]
      }
   ]
}
```

Resource takes the form `arn:aws:rds-db:region:account-id:dbuser:dbi-resource-id/database-user-name`

database-user-name is the name of the MySQL database account to associate with IAM authentication. In the example policy, the database account is david_lu.

With IAM database authentication, we don't need to assign database passwords to the MySQL user accounts we create. Instead, authentication is handled by AWSAuthenticationPlugin—an AWS-provided plugin that works seamlessly with IAM to authenticate our IAM users.

## For MySQL

To create a database account for MySQL, connect to the DB instance or DB cluster and issue the CREATE USER statement, as shown in the following example. You can additionally narrow the grants down a bit.

```
CREATE USER 'iam_sre' IDENTIFIED WITH AWSAuthenticationPlugin as 'RDS';
GRANT SELECT ON sre.* TO 'iam_sre'@'%';
FLUSH PRIVILEGES;
```

The IDENTIFIED WITH clause allows MySQL to use the AWSAuthenticationPlugin to authenticate the database account (david_lu). The AS 'RDS' clause maps the jane_doe database account to the corresponding IAM user or role.

## For Postgres

To create a database account for Postgres, connect to the DB instance or DB cluster and issue the CREATE USER / Role statement, as shown in the following example. You can additionally narrow the grants down a bit.

```
CREATE ROLE iam_sre WITH LOGIN;
GRANT rds_iam TO iam_sre;
```

To connect the new created DB instance from CLI:

```
export RDSHOST="sre-testing.xxxxxxx.ap-southeast-2.rds.amazonaws.com"
 
export PGPASSWORD="$(aws rds generate-db-auth-token --hostname $RDSHOST --port 5432 --region ap-southeast-2 --username iam_sre)"
 
psql "host=$RDSHOST port=5432 dbname=sre user=iam_sre"
```

With IAM database authentication, we use an authentication token when we connect to a DB instance or DB cluster. An authentication token is a string of characters that we use instead of a password. Once we generate an authentication token, it's valid for `15` minutes before it expires. If we try to connect using an expired token, the connection request is denied.

Every authentication token must be accompanied by a valid signature, using AWS signature version 4

We can connect from the command line to an RDS DB instance or Aurora DB cluster with the AWS CLI and mysql command line tool as described following.

# References

http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.Enabling.html
http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.IAMPolicy.html
https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.DBAccounts.html
https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.Connecting.html
