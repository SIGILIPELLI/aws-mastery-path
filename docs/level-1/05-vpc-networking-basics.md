# 05 · VPC Networking Basics

A VPC (Virtual Private Cloud) is your own isolated slice of AWS network —
every EC2 instance, RDS database, and Lambda function attached to a VPC
lives inside one. New accounts get a **default VPC** in every region
(pre-wired with public subnets), which is why Module 3's EC2 instance worked
without you configuring any networking — it landed in the default VPC. This
module opens up what was happening underneath, and builds a custom VPC with
public and private subnets, which the RDS and capstone modules will use.

## Core concepts

| Concept | What it is |
|---|---|
| **VPC** | A private IP address range (CIDR block) you control, isolated from other customers' networks. |
| **Subnet** | A subdivision of the VPC's CIDR, tied to one Availability Zone. |
| **Public subnet** | Has a route to an Internet Gateway — resources here can have public IPs. |
| **Private subnet** | No direct route to the internet — used for databases and internal services. |
| **Internet Gateway (IGW)** | Attached to the VPC; the door to/from the public internet. |
| **Route table** | Rules that decide where a subnet's traffic goes, based on destination CIDR. |
| **NAT Gateway** | Lets private-subnet resources make *outbound* internet calls without being reachable from outside. |

## Create a VPC and subnets

```bash
# VPC with a /16 CIDR (65,536 addresses)
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=training-vpc}]'
# VpcId: vpc-0123456789abcdef0

# Public subnet, /24 (256 addresses), in AZ us-east-1a
aws ec2 create-subnet \
  --vpc-id vpc-0123456789abcdef0 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=training-public-subnet}]'
# SubnetId: subnet-0111111111111111

# Private subnet, /24, same AZ
aws ec2 create-subnet \
  --vpc-id vpc-0123456789abcdef0 \
  --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=training-private-subnet}]'
# SubnetId: subnet-0222222222222222
```

## Attach an Internet Gateway

```bash
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=training-igw}]'
# InternetGatewayId: igw-0123456789abcdef0

aws ec2 attach-internet-gateway \
  --vpc-id vpc-0123456789abcdef0 \
  --internet-gateway-id igw-0123456789abcdef0
```

## Route tables: making the public subnet actually public

A subnet is only "public" because its route table sends `0.0.0.0/0` (all
non-local traffic) to the Internet Gateway. Nothing about a subnet is
inherently public or private — it's entirely a function of its route table.

```bash
aws ec2 create-route-table \
  --vpc-id vpc-0123456789abcdef0 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=training-public-rt}]'
# RouteTableId: rtb-0123456789abcdef0

# Send all non-local traffic to the Internet Gateway
aws ec2 create-route \
  --route-table-id rtb-0123456789abcdef0 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0123456789abcdef0

# Associate the route table with the public subnet
aws ec2 associate-route-table \
  --route-table-id rtb-0123456789abcdef0 \
  --subnet-id subnet-0111111111111111
```

The private subnet, left on the VPC's default (main) route table with no
`0.0.0.0/0` route, has no path to the internet at all — which is exactly
what you want for a database subnet.

## Security groups vs. Network ACLs

Both filter traffic, but at different layers and with different semantics:

| | Security Group | Network ACL (NACL) |
|---|---|---|
| Attaches to | Individual instance/ENI | Entire subnet |
| Rule type | **Allow only** | Allow **and** deny |
| State | Stateful (return traffic auto-allowed) | Stateless (must allow both directions explicitly) |
| Evaluation | All rules evaluated, most permissive wins | Rules evaluated **in numbered order**, first match wins |
| Default | Every VPC's default SG denies all inbound | Every VPC's default NACL allows all traffic |

In practice: **security groups are your everyday tool** (instance-level,
simple allow-list, used constantly). **NACLs are a coarser, subnet-wide
backstop** — most teams leave the default "allow all" NACL alone and do all
their real access control with security groups, reaching for a NACL only
when they need to explicitly *block* a specific CIDR at the subnet level
(something security groups can't do, since they have no deny rules).

```bash
# Example: NACL rule explicitly blocking a specific bad CIDR from the whole subnet
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0123456789abcdef0 \
  --rule-number 100 \
  --protocol -1 \
  --rule-action deny \
  --egress false \
  --cidr-block 198.51.100.0/24
```

## Auto-assigning public IPs in the public subnet

```bash
aws ec2 modify-subnet-attribute \
  --subnet-id subnet-0111111111111111 \
  --map-public-ip-on-launch
```

With this set, any instance launched into this subnet automatically gets a
public IP — matching the default-VPC behavior you relied on in Module 3.

## Putting it together

```
                 ┌───────────────── VPC 10.0.0.0/16 ─────────────────┐
                 │                                                    │
   Internet ───IGW── public subnet 10.0.1.0/24 ── EC2 (public IP)     │
                 │         │                                          │
                 │    route table: 0.0.0.0/0 → IGW                    │
                 │                                                    │
                 │   private subnet 10.0.2.0/24 ── RDS (no public IP) │
                 │         │                                          │
                 │    route table: local only (no internet route)     │
                 └────────────────────────────────────────────────────┘
```

This is the exact shape the RDS module (next) and the capstone use: public
subnet for anything that needs direct internet access, private subnet for
the database.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws ec2 create-vpc --cidr-block CIDR` | Create a VPC. |
| `aws ec2 create-subnet --vpc-id ID --cidr-block CIDR --availability-zone AZ` | Create a subnet. |
| `aws ec2 create-internet-gateway` / `attach-internet-gateway` | Create and attach an IGW to a VPC. |
| `aws ec2 create-route-table --vpc-id ID` | Create a route table. |
| `aws ec2 create-route --route-table-id ID --destination-cidr-block 0.0.0.0/0 --gateway-id IGW` | Route all external traffic to the IGW. |
| `aws ec2 associate-route-table --route-table-id ID --subnet-id ID` | Bind a route table to a subnet. |
| `aws ec2 modify-subnet-attribute --subnet-id ID --map-public-ip-on-launch` | Auto-assign public IPs in a subnet. |
| `aws ec2 describe-vpcs` / `describe-subnets` / `describe-route-tables` | Inspect what you've built. |

## Exercise

1. Create a VPC (`10.0.0.0/16`) with one public subnet (`10.0.1.0/24`) and
   one private subnet (`10.0.2.0/24`) in the same Availability Zone.
2. Attach an Internet Gateway and wire up a route table so only the public
   subnet can reach `0.0.0.0/0`.
3. Enable auto-assign public IP on the public subnet.
4. Launch a `t3.micro` instance into the public subnet (reusing the key pair
   and security group from Module 3) and confirm you can still SSH into it.
5. Run `aws ec2 describe-route-tables --filters
   "Name=vpc-id,Values=<your-vpc-id>"` and identify, from the output alone,
   which route table is associated with which subnet and why one has
   internet access and the other doesn't.
