# 03 · EC2 Compute

EC2 (Elastic Compute Cloud) rents you virtual machines — "instances" — by
the second. It's the most fundamental compute building block in AWS: almost
every other compute service (ECS, EKS, even parts of Lambda's underlying
infrastructure) ultimately runs on EC2 under the hood. This module covers
launching an instance from the CLI, the security concepts that gate access
to it (key pairs, security groups), choosing an image and size, and
connecting to it over SSH.

## Core concepts

| Concept | What it is |
|---|---|
| **AMI** (Amazon Machine Image) | A template: OS + pre-installed software an instance boots from. |
| **Instance type** | The hardware profile (vCPUs, memory, network) — e.g. `t3.micro`, `m5.large`. |
| **Key pair** | An SSH public/private key pair; AWS installs the public half on the instance, you keep the private half. |
| **Security group** | A virtual firewall attached to the instance, controlling inbound/outbound traffic by port and source. |
| **Elastic IP** | An optional static public IP you can attach to an instance (regular public IPs change if you stop/start). |

## Create a key pair

```bash
aws ec2 create-key-pair \
  --key-name training-key \
  --query "KeyMaterial" \
  --output text > training-key.pem

chmod 400 training-key.pem
```

`chmod 400` is required — SSH refuses to use a private key file that other
users on your machine could read. **There is no way to retrieve this private
key again from AWS** if you lose it; you'd need to create a new key pair.

## Create a security group

```bash
aws ec2 create-security-group \
  --group-name training-sg \
  --description "SSH and HTTP for training instances"
# {
#     "GroupId": "sg-0123456789abcdef0"
# }

# Allow SSH only from your own IP (safer than 0.0.0.0/0)
MY_IP=$(curl -s https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp --port 22 \
  --cidr "${MY_IP}/32"

# Allow HTTP from anywhere (for a web server you'll test in this module)
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp --port 80 \
  --cidr 0.0.0.0/0
```

Security groups are **stateful**: a response to an allowed inbound request
is automatically allowed back out, so you rarely need explicit outbound
rules for typical request/response traffic.

## Choose an AMI and instance type

```bash
# Find the latest Amazon Linux 2023 AMI for your region
aws ssm get-parameters \
  --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query "Parameters[0].Value" --output text
# ami-0abcdef1234567890
```

Common instance types you'll see in this course:

| Type | vCPU | Memory | Typical use | Free tier? |
|---|---|---|---|---|
| `t3.micro` | 2 | 1 GiB | Small dev/test workloads | Yes (750 hrs/month for 12 months on new accounts) |
| `t3.small` | 2 | 2 GiB | Light web apps | No |
| `m5.large` | 2 | 8 GiB | General-purpose production | No |

Stick to `t3.micro` for this course to stay in the free tier.

## Launch an instance

```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --key-name training-key \
  --security-group-ids sg-0123456789abcdef0 \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=training-instance}]'
```

The response includes an `InstanceId` (e.g. `i-0123456789abcdef0`). Poll its
state:

```bash
aws ec2 describe-instances \
  --instance-ids i-0123456789abcdef0 \
  --query "Reservations[0].Instances[0].[State.Name,PublicIpAddress]" \
  --output table

# Or wait for it to become reachable before connecting
aws ec2 wait instance-status-ok --instance-ids i-0123456789abcdef0
```

## Connect via SSH

```bash
ssh -i training-key.pem ec2-user@<PublicIpAddress>
```

`ec2-user` is the default login for Amazon Linux AMIs (Ubuntu AMIs use
`ubuntu`). Once connected, this is a normal Linux box:

```bash
# On the instance
sudo yum install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd
echo "<h1>Hello from EC2</h1>" | sudo tee /var/www/html/index.html
```

Visit `http://<PublicIpAddress>` in a browser — you should see "Hello from
EC2" (this works because you opened port 80 to `0.0.0.0/0` in the security
group above).

## Attaching an IAM role to an instance

Instead of putting access keys on the instance, attach the instance profile
you created in the previous module so it can talk to AWS APIs (e.g. read S3)
using temporary, auto-rotating credentials:

```bash
aws ec2 associate-iam-instance-profile \
  --instance-id i-0123456789abcdef0 \
  --iam-instance-profile Name=EC2-S3-ReadOnly-Profile
```

From inside the instance, any AWS SDK or the CLI automatically picks up
these credentials via the instance metadata service — no `aws configure`
needed on the instance itself.

## Stopping vs. terminating

| Action | What happens | Billing |
|---|---|---|
| **Stop** | Instance shuts down, disk (EBS) persists, public IP is released (unless Elastic IP) | Stops compute billing; storage still billed |
| **Terminate** | Instance and its root EBS volume are deleted (unless "delete on termination" is disabled) | Stops all billing for this instance |

```bash
aws ec2 stop-instances --instance-ids i-0123456789abcdef0
aws ec2 start-instances --instance-ids i-0123456789abcdef0
aws ec2 terminate-instances --instance-ids i-0123456789abcdef0
```

!!! warning "Clean up when you're done"
    A running `t3.micro` is free-tier-eligible for the first 12 months on a
    new account, but only up to 750 hours/month total — and only while
    within that window. Terminate instances you're not actively using so you
    don't get a surprise bill.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws ec2 create-key-pair --key-name NAME` | Create an SSH key pair, prints the private key. |
| `aws ec2 create-security-group --group-name NAME --description DESC` | Create a firewall group. |
| `aws ec2 authorize-security-group-ingress --group-id ID --protocol tcp --port N --cidr CIDR` | Open a port. |
| `aws ec2 run-instances --image-id AMI --instance-type TYPE --key-name KEY --security-group-ids SG` | Launch an instance. |
| `aws ec2 describe-instances` | List/describe instances. |
| `aws ec2 wait instance-status-ok --instance-ids ID` | Block until an instance passes health checks. |
| `ssh -i KEY.pem ec2-user@IP` | Connect to an Amazon Linux instance. |
| `aws ec2 stop-instances` / `start-instances` | Pause/resume billing for compute (storage still billed). |
| `aws ec2 terminate-instances` | Delete the instance and its root volume. |

## Exercise

1. Launch a `t3.micro` Amazon Linux instance with a security group that
   allows SSH from your IP and HTTP from anywhere.
2. SSH in, install and start `httpd`, and serve a custom HTML page with your
   name on it.
3. Confirm you can view the page from your browser via the instance's public
   IP.
4. Attach the `EC2-S3-ReadOnly-Profile` instance profile you created in the
   IAM module, then from inside the instance run
   `aws s3 ls` (install the CLI on the instance first if needed) — confirm
   it works with **no** `aws configure` step, proving the role is supplying
   credentials.
5. Terminate the instance when you're done so it stops counting against your
   free-tier hours.
