# 02 · Auto Scaling & Load Balancing

A single EC2 instance (Level 1, module 3) is a single point of failure and
a fixed amount of capacity. This module fixes both problems: an **Auto
Scaling Group (ASG)** keeps a fleet of identical instances running and
replaces any that fail, scaling the fleet up or down with demand; an
**Application Load Balancer (ALB)** spreads incoming traffic across that
fleet and stops routing to any instance that fails a health check.
Together they're the standard pattern for a resilient, elastic web tier
before you reach for ECS (previous module) or Beanstalk (next module).

## Core concepts

| Concept | What it is |
|---|---|
| **Launch template** | A reusable blueprint (AMI, instance type, security groups, user data) instances are created from. |
| **Auto Scaling Group (ASG)** | Maintains a fleet between min/max size, targeting a desired capacity, across chosen subnets/AZs. |
| **Scaling policy** | The rule that adjusts desired capacity — e.g. target tracking on average CPU. |
| **Target group** | A named set of registered targets (instances) the load balancer routes to, plus its health check config. |
| **Listener** | A rule on the load balancer (e.g. "port 80, HTTP") that forwards matching requests to a target group. |
| **Health check** | A periodic probe (HTTP path, expected status) — targets failing it stop receiving traffic. |

## Create a launch template

```bash
cat > user-data.sh << 'EOF'
#!/bin/bash
yum install -y httpd
systemctl enable httpd
echo "<h1>Served by $(hostname -f)</h1>" > /var/www/html/index.html
systemctl start httpd
EOF

aws ec2 create-launch-template \
  --launch-template-name training-lt \
  --version-description "v1" \
  --launch-template-data "{
    \"ImageId\": \"ami-0abcdef1234567890\",
    \"InstanceType\": \"t3.micro\",
    \"SecurityGroupIds\": [\"sg-0123456789abcdef0\"],
    \"UserData\": \"$(base64 -i user-data.sh)\"
  }"
# LaunchTemplateId: lt-0123456789abcdef0
```

`UserData` must be base64-encoded — the CLI does not do this for you when
it's embedded in a JSON string like above (only `--user-data file://` on
some commands auto-encodes). Each instance the ASG launches runs this
script once at boot, installing and starting a web server without you
logging into it by hand.

## Create the Application Load Balancer

```bash
aws elbv2 create-load-balancer \
  --name training-alb \
  --subnets subnet-0aaa1111 subnet-0bbb2222 \
  --security-groups sg-0123456789abcdef0 \
  --scheme internet-facing --type application
# LoadBalancerArn: arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/training-alb/50dc6c495c0c9188
# DNSName: training-alb-123456789.us-east-1.elb.amazonaws.com

aws elbv2 create-target-group \
  --name training-tg \
  --protocol HTTP --port 80 \
  --vpc-id vpc-0123456789abcdef0 \
  --health-check-path /
# TargetGroupArn: arn:...:targetgroup/training-tg/6d0ecf831eec9f09

aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/training-alb/50dc6c495c0c9188 \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:...:targetgroup/training-tg/6d0ecf831eec9f09
```

An ALB **requires at least two subnets in two different Availability
Zones** — this is enforced at creation time, not just a best practice, so
a single-AZ VPC will reject the `create-load-balancer` call outright.

## Create the Auto Scaling Group, attached to the target group

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name training-asg \
  --launch-template "LaunchTemplateName=training-lt,Version=\$Latest" \
  --min-size 2 --max-size 6 --desired-capacity 2 \
  --vpc-zone-identifier "subnet-0aaa1111,subnet-0bbb2222" \
  --target-group-arns arn:...:targetgroup/training-tg/6d0ecf831eec9f09 \
  --health-check-type ELB \
  --health-check-grace-period 60
```

`--health-check-type ELB` (instead of the default `EC2`) tells the ASG to
trust the target group's health check, not just "is the instance
running." `--health-check-grace-period` gives each new instance time to
boot and start the web server before it can be marked unhealthy and
replaced — set it at least as long as your slowest cold start.

## Add a target-tracking scaling policy

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name training-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 50.0
  }'
```

This tells the ASG "keep average CPU across the fleet near 50%" — it adds
instances when the fleet is busier than that and removes them when it's
idler, within the `min-size`/`max-size` bounds. There's a built-in
**cooldown** between scaling actions so it doesn't thrash on brief spikes.

## Check health and scale manually

```bash
aws elbv2 describe-target-health \
  --target-group-arn arn:...:targetgroup/training-tg/6d0ecf831eec9f09 \
  --query "TargetHealthDescriptions[].[Target.Id,TargetHealth.State]" \
  --output table
# ---------------------------------------
# |  i-0123456789abcdef0 |  healthy      |
# |  i-0fedcba9876543210 |  healthy      |

aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names training-asg \
  --query "AutoScalingGroups[0].[DesiredCapacity,MinSize,MaxSize]"

# Manually override desired capacity (e.g. for a planned traffic spike)
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name training-asg --desired-capacity 4
```

Visit the ALB's `DNSName` in a browser and refresh a few times — you
should see the hostname in the response change as the ALB round-robins
across healthy instances.

## Instance refresh (rolling replace on a template change)

```bash
# After creating a new launch template version with an updated AMI/user data
aws ec2 create-launch-template-version \
  --launch-template-name training-lt \
  --source-version 1 \
  --launch-template-data '{"ImageId": "ami-0newamiid000000000"}'

aws autoscaling start-instance-refresh \
  --auto-scaling-group-name training-asg \
  --preferences '{"MinHealthyPercentage": 50, "InstanceWarmup": 60}'
```

Instance refresh replaces instances gradually while keeping at least
`MinHealthyPercentage` of the fleet in service — the ASG equivalent of the
rolling deployment ECS does automatically (previous module).

!!! warning "The ALB itself is billed hourly plus per LCU, whether or not it has healthy targets"
    An idle ALB with zero traffic still accrues an hourly charge and a
    small baseline **LCU** (Load Balancer Capacity Unit) charge — deleting
    it, not just scaling the ASG to zero, is what stops that cost during
    cleanup.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws ec2 create-launch-template` | Define the AMI/instance type/security groups/user data for future instances. |
| `aws elbv2 create-load-balancer --type application --subnets ...` | Create an ALB across 2+ AZs. |
| `aws elbv2 create-target-group --health-check-path P` | Define a routable, health-checked target set. |
| `aws elbv2 create-listener --default-actions Type=forward,TargetGroupArn=...` | Route incoming traffic to a target group. |
| `aws autoscaling create-auto-scaling-group --target-group-arns ...` | Create a self-healing fleet wired to the ALB. |
| `aws autoscaling put-scaling-policy --policy-type TargetTrackingScaling` | Scale automatically on a metric target. |
| `aws elbv2 describe-target-health` | Check which instances are passing health checks. |
| `aws autoscaling start-instance-refresh` | Roll out a new launch template version gradually. |

## Exercise

1. Create a launch template with user data that installs and starts a web
   server showing the instance's hostname.
2. Create an ALB (2+ AZs), a target group with an HTTP health check on
   `/`, and a listener forwarding port 80 to it.
3. Create an ASG (`min 2`, `max 6`, `desired 2`) attached to the target
   group, with `health-check-type ELB` and a sensible grace period.
4. Confirm both instances show `healthy` in `describe-target-health`, then
   refresh the ALB's DNS name in a browser several times and observe the
   hostname changing.
5. Add a target-tracking scaling policy on `ASGAverageCPUUtilization` at
   50%, then generate load on one instance (e.g. `stress-ng` or a busy
   loop over SSH) and watch `describe-auto-scaling-groups` show desired
   capacity increase.
6. Tear down in order — delete the ASG (this terminates its instances),
   then the listener, target group, and load balancer — so nothing keeps
   billing.
