# 06 · Elastic Beanstalk

Module 2 built an EC2 fleet, ASG, and ALB by hand — several separate
resources you wired together yourself. **Elastic Beanstalk** is a
higher-level service that provisions and manages that same combination
(EC2, ASG, ALB, security groups, and optionally RDS) from a single
application upload, while still giving you access to the underlying
resources if you need to tune them. It sits between "fully manual
infrastructure" and "fully abstracted" (Lambda/ECS Fargate) — you deploy
code, Beanstalk manages the servers.

## Core concepts

| Concept | What it is |
|---|---|
| **Application** | A logical container for your project's environments and versions. |
| **Application version** | One uploaded, deployable build (a zip in S3) — Beanstalk keeps a history of these. |
| **Environment** | A running deployment of one application version, with its own EC2/ASG/ALB resources. |
| **Platform** | The managed runtime stack (e.g. "Python 3.12 on Amazon Linux 2023", "Docker") Beanstalk provisions for you. |
| **`.ebextensions`** | Config files in your app bundle that customize the environment's resources (extra config, packages, resources). |
| **Environment tier** | Web server (handles HTTP traffic) vs. worker (processes an SQS queue in the background). |

## Create the application

```bash
aws elasticbeanstalk create-application \
  --application-name training-app \
  --description "Training web app"
```

## Package and upload a version

```bash
zip -r app.zip . -x ".git/*"

aws s3 mb s3://training-eb-versions-2026
aws s3 cp app.zip s3://training-eb-versions-2026/app-v1.zip

aws elasticbeanstalk create-application-version \
  --application-name training-app \
  --version-label v1 \
  --source-bundle S3Bucket=training-eb-versions-2026,S3Key=app-v1.zip
```

The zip's root must contain what the platform expects to run (for
Python, typically an `application.py`/`requirements.txt`; check the
platform's docs) — Beanstalk unpacks it onto each instance it launches.

## Find a platform and create the environment

```bash
aws elasticbeanstalk list-available-solution-stacks \
  --query "SolutionStacks[?contains(@, 'Python 3.12')]"
# "64bit Amazon Linux 2023 v4.x.x running Python 3.12"

aws elasticbeanstalk create-environment \
  --application-name training-app \
  --environment-name training-app-prod \
  --solution-stack-name "64bit Amazon Linux 2023 v4.x.x running Python 3.12" \
  --version-label v1 \
  --option-settings \
      Namespace=aws:autoscaling:asg,OptionName=MinSize,Value=2 \
      Namespace=aws:autoscaling:asg,OptionName=MaxSize,Value=4 \
      Namespace=aws:ec2:instances,OptionName=InstanceTypes,Value=t3.micro
```

Under the hood this single call creates almost exactly what module 2
built by hand: a launch template, an ASG with the given min/max, an ALB,
target group, and listener — but as one managed unit Beanstalk tracks and
lets you update together.

## Check status and get the URL

```bash
aws elasticbeanstalk describe-environments \
  --environment-names training-app-prod \
  --query "Environments[0].[Status,Health,CNAME]"
# ["Ready", "Green", "training-app-prod.eba-abc123.us-east-1.elasticbeanstalk.com"]
```

`Status` (`Launching`/`Updating`/`Ready`/`Terminating`) is about the
environment's lifecycle; `Health` (`Green`/`Yellow`/`Red`) is about
whether the running app is actually responding correctly — an environment
can be `Ready` and `Red` at the same time if the deployed code is broken.

## Deploying an update

```bash
zip -r app.zip . -x ".git/*"
aws s3 cp app.zip s3://training-eb-versions-2026/app-v2.zip

aws elasticbeanstalk create-application-version \
  --application-name training-app \
  --version-label v2 \
  --source-bundle S3Bucket=training-eb-versions-2026,S3Key=app-v2.zip

aws elasticbeanstalk update-environment \
  --environment-name training-app-prod \
  --version-label v2
```

Beanstalk performs a **rolling update** across the ASG by default
(configurable to immutable or all-at-once) — new instances get `v2`,
old ones are drained, similar in spirit to the ECS and ASG rolling
deployments in earlier modules, but orchestrated for you.

## Customizing the environment with `.ebextensions`

```yaml
# .ebextensions/01-environment.config
option_settings:
  aws:elasticbeanstalk:application:environment:
    LOG_LEVEL: info
  aws:elasticbeanstalk:environment:proxy:
    ProxyServer: nginx

Resources:
  AWSEBAutoScalingScalingPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AdjustmentType: ChangeInCapacity
      ScalingAdjustment: 1
      Cooldown: 60
```

Files under `.ebextensions/` in your app bundle are YAML/JSON and are
applied every time the environment is created or updated — this is how
you set environment variables, install OS packages, or (as above) attach
extra CloudFormation resources Beanstalk doesn't expose as a plain option,
since every Beanstalk environment is itself backed by a CloudFormation
stack you can inspect.

## Blue/green deploys via CNAME swap

```bash
aws elasticbeanstalk create-environment \
  --application-name training-app \
  --environment-name training-app-green \
  --solution-stack-name "64bit Amazon Linux 2023 v4.x.x running Python 3.12" \
  --version-label v2

# After validating training-app-green independently:
aws elasticbeanstalk swap-environment-cnames \
  --source-environment-name training-app-prod \
  --destination-environment-name training-app-green
```

This swaps the two environments' public CNAMEs atomically — traffic moves
to the fully-tested new environment instantly, and rolling back is just
swapping the CNAMEs back, rather than re-deploying old code under time
pressure.

!!! warning "Beanstalk itself is free — you pay for the EC2/ALB/RDS it creates"
    There's no separate Beanstalk service fee, but every environment
    creates real, billable EC2 instances and an ALB (and RDS if you attach
    a database) exactly as if you'd created them yourself in module 2 — a
    forgotten `training-app-prod` environment costs the same as a
    forgotten hand-built ASG + ALB.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws elasticbeanstalk create-application --application-name N` | Create the logical app container. |
| `aws elasticbeanstalk create-application-version --source-bundle S3Bucket=B,S3Key=K` | Register a deployable build. |
| `aws elasticbeanstalk create-environment --solution-stack-name S --version-label V` | Provision a running environment from a version. |
| `aws elasticbeanstalk describe-environments` | Check `Status`/`Health`/`CNAME`. |
| `aws elasticbeanstalk update-environment --version-label V` | Roll out a new version to an existing environment. |
| `aws elasticbeanstalk swap-environment-cnames` | Atomically swap two environments' public URLs (blue/green). |
| `aws elasticbeanstalk terminate-environment --environment-name N` | Tear down an environment's underlying resources. |

## Exercise

1. Create an application and upload a first version of a small web app as
   a zip.
2. Create a web-tier environment on a matching platform, with `MinSize: 2`
   in `.option-settings`, and confirm `describe-environments` shows
   `Status: Ready`, `Health: Green`.
3. Add a `.ebextensions/01-environment.config` setting an environment
   variable, redeploy, and confirm your app can read it.
4. Deploy a `v2` with a visible change, and watch `describe-environments`
   `Health` during the rolling update.
5. Create a second `-green` environment on `v2`, validate it independently
   via its own `CNAME`, then swap CNAMEs with the production environment
   and confirm the swap is instant from the original URL.
6. Terminate both environments (and delete the S3 version bucket) when
   done — this removes the underlying EC2/ASG/ALB resources they created.
