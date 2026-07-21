# 06 · RDS Databases

RDS (Relational Database Service) gives you a managed relational database —
MySQL, PostgreSQL, MariaDB, SQL Server, or Oracle — where AWS handles patching,
backups, and failover, so you focus on schema and queries instead of database
administration. This module covers creating a small RDS instance inside the
private subnet from the previous module, securing it with a security group
that only allows connections from your EC2 instance, and connecting to it.

## Why RDS instead of installing MySQL on EC2 yourself

You *could* install PostgreSQL on an EC2 instance directly, but RDS handles,
automatically and with no extra setup: nightly backups with point-in-time
recovery, minor version patching, Multi-AZ failover (a synchronously
replicated standby in another Availability Zone), and CloudWatch metrics out
of the box. The tradeoff is less low-level control — you can't SSH into an
RDS instance's underlying OS.

## Networking prerequisite: a DB subnet group

RDS instances live in the private subnet(s) from the previous module. RDS
requires at least two subnets, in two different Availability Zones, grouped
into a **DB subnet group** — even for a single-AZ instance — so create a
second private subnet first:

```bash
aws ec2 create-subnet \
  --vpc-id vpc-0123456789abcdef0 \
  --cidr-block 10.0.3.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=training-private-subnet-2}]'
# SubnetId: subnet-0333333333333333

aws rds create-db-subnet-group \
  --db-subnet-group-name training-db-subnet-group \
  --db-subnet-group-description "Private subnets for RDS" \
  --subnet-ids subnet-0222222222222222 subnet-0333333333333333
```

## Security group: only allow EC2 to connect

```bash
aws ec2 create-security-group \
  --group-name training-db-sg \
  --description "Allow Postgres only from the app security group" \
  --vpc-id vpc-0123456789abcdef0
# GroupId: sg-0dbdbdbdbdbdbdbdb

# Allow port 5432 only from instances in your existing EC2 security group,
# not from any IP -- this is the standard "app tier talks to data tier" pattern
aws ec2 authorize-security-group-ingress \
  --group-id sg-0dbdbdbdbdbdbdbdb \
  --protocol tcp --port 5432 \
  --source-group sg-0123456789abcdef0
```

Note the `--source-group` instead of `--cidr` — this rule says "allow
traffic from anything wearing the `training-sg` security group", which
automatically covers every current and future EC2 instance using it,
without ever hard-coding an IP.

## Create the RDS instance

```bash
aws rds create-db-instance \
  --db-instance-identifier training-postgres \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 16.3 \
  --master-username trainingadmin \
  --master-user-password 'ChangeMe123!' \
  --allocated-storage 20 \
  --db-subnet-group-name training-db-subnet-group \
  --vpc-security-group-ids sg-0dbdbdbdbdbdbdbdb \
  --no-publicly-accessible \
  --backup-retention-period 1
```

`db.t3.micro` with 20 GiB of storage is free-tier eligible for 12 months.
`--no-publicly-accessible` keeps it reachable only from inside the VPC —
correct for a real app, since there's no reason a database should have a
public IP at all.

Wait for it to become available (this typically takes several minutes):

```bash
aws rds wait db-instance-available --db-instance-identifier training-postgres

aws rds describe-db-instances \
  --db-instance-identifier training-postgres \
  --query "DBInstances[0].Endpoint" --output json
# {
#     "Address": "training-postgres.abc123xyz.us-east-1.rds.amazonaws.com",
#     "Port": 5432
# }
```

## Connect from EC2

Because the database is private, you connect *from* the EC2 instance you
launched earlier (which must be in the same VPC and carry the `training-sg`
security group):

```bash
# On the EC2 instance
sudo yum install -y postgresql15

psql -h training-postgres.abc123xyz.us-east-1.rds.amazonaws.com \
     -p 5432 -U trainingadmin -d postgres
# Password for user trainingadmin: ********

postgres=> CREATE DATABASE trainingdb;
postgres=> \c trainingdb
trainingdb=> CREATE TABLE notes (id SERIAL PRIMARY KEY, body TEXT);
trainingdb=> INSERT INTO notes (body) VALUES ('hello from RDS');
trainingdb=> SELECT * FROM notes;
#  id |      body
# ----+-----------------
#   1 | hello from RDS
```

If the connection hangs rather than immediately failing, it's almost always
a security-group issue (wrong source group, or connecting from an instance
outside the allowed group) — check that first.

## Automated backups and point-in-time recovery

RDS takes automated daily backups (retained for `--backup-retention-period`
days) plus continuous transaction logs, letting you restore to any point in
time within that window — not just to the last backup:

```bash
# Restore to a brand-new instance as of a specific point in time
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier training-postgres \
  --target-db-instance-identifier training-postgres-restored \
  --restore-time 2026-07-20T10:00:00Z
```

A restore always creates a **new** instance — it never overwrites the
original in place, which is a deliberate safety property.

## Cleaning up

RDS instances are billed by the hour they're running, regardless of
traffic — unlike Lambda, there's no free "idle" state. Delete instances you
aren't using:

```bash
aws rds delete-db-instance \
  --db-instance-identifier training-postgres \
  --skip-final-snapshot
```

`--skip-final-snapshot` is appropriate for throwaway training data; for
anything real, omit it (or pass `--final-db-snapshot-identifier NAME`) so
RDS takes one last snapshot before deleting.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws rds create-db-subnet-group` | Group subnets (2+ AZs) for RDS to place instances in. |
| `aws rds create-db-instance --engine postgres ...` | Provision a managed database instance. |
| `aws rds wait db-instance-available` | Block until the instance is ready. |
| `aws rds describe-db-instances` | Get endpoint address, port, status. |
| `aws rds restore-db-instance-to-point-in-time` | Restore to a new instance as of a past timestamp. |
| `aws rds modify-db-instance` | Change instance class, storage, etc. |
| `aws rds delete-db-instance --skip-final-snapshot` | Delete the instance (no final backup). |
| `--source-group` (on a security group rule) | Allow traffic only from members of another security group. |

## Exercise

1. Create a second private subnet in a different AZ, a DB subnet group using
   both private subnets, and a `training-db-sg` security group that only
   allows port 5432 from your EC2 security group.
2. Launch a `db.t3.micro` PostgreSQL instance, non-publicly-accessible, using
   that subnet group and security group.
3. From your EC2 instance, install a PostgreSQL client, connect to the RDS
   endpoint, create a table, and insert a row.
4. Confirm that attempting to connect to the RDS endpoint from your own
   laptop (outside the VPC) fails or hangs — proving the private-subnet +
   security-group setup is actually enforcing the boundary.
5. Delete the RDS instance with `--skip-final-snapshot` when finished, and
   confirm with `describe-db-instances` that it no longer appears (or shows
   status `deleting`).
