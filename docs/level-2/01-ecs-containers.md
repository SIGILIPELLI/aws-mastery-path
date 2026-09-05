# 01 · Containers on AWS (ECS/Fargate)

Level 1 ran your code on EC2 instances (module 3) and inside Lambda
(module 7). Containers sit between those two worlds: you package an app
with everything it needs into a portable image, then hand that image to a
scheduler that runs it for you. **ECS** (Elastic Container Service) is
AWS's native container orchestrator, and the **Fargate** launch type runs
your containers without you provisioning or patching any underlying EC2
instances — you specify vCPU/memory per task and AWS handles the host.

## Core concepts

| Concept | What it is |
|---|---|
| **Image** | A packaged filesystem + startup command (built with `docker build`), stored in a registry. |
| **ECR** | Elastic Container Registry — AWS's private Docker registry. |
| **Task definition** | A JSON blueprint: which image(s) to run, CPU/memory, ports, IAM roles, logging. |
| **Task** | A running instance of a task definition — one or more containers scheduled together. |
| **Cluster** | A logical grouping of tasks/services (with Fargate, mostly a namespace — no servers to manage). |
| **Service** | Keeps a desired number of tasks running, replacing any that die, and can register tasks with a load balancer. |
| **Launch type** | `FARGATE` (serverless, per-task billing) vs `EC2` (you manage the underlying instances). |

## Build and push an image to ECR

```bash
# Build locally (Dockerfile in the current directory)
docker build -t training-app:latest .

# Create a private repository
aws ecr create-repository --repository-name training-app
# {
#     "repository": {
#         "repositoryUri": "123456789012.dkr.ecr.us-east-1.amazonaws.com/training-app"
#     }
# }

# Authenticate Docker to ECR, then tag and push
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

docker tag training-app:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/training-app:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/training-app:latest
```

The `get-login-password` token is valid for 12 hours — CI pipelines
re-authenticate on every run rather than caching it.

## Create a cluster

```bash
aws ecs create-cluster --cluster-name training-cluster
```

With Fargate, a "cluster" creates no billable infrastructure by itself —
it's purely a grouping construct. You only pay once tasks are running
inside it.

## Define a task

Save this as `task-def.json`:

```json
{
  "family": "training-app",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "training-app",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/training-app:latest",
      "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/training-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

```bash
aws logs create-log-group --log-group-name /ecs/training-app

aws ecs register-task-definition --cli-input-json file://task-def.json
# revision: 1
```

Fargate requires `awsvpc` network mode — each task gets its own elastic
network interface (its own private IP), unlike the shared host networking
EC2-launch-type tasks can use.

## Execution role vs. task role

Two different IAM roles are easy to confuse:

| Role | Used for | Example permission |
|---|---|---|
| **Execution role** | Actions ECS takes *on your behalf* to start the task | Pull the image from ECR, write logs to CloudWatch |
| **Task role** | Actions *your application code* takes at runtime | Read an S3 bucket, write to DynamoDB |

`executionRoleArn` above only needs `AmazonECSTaskExecutionRolePolicy`. If
your app talks to other AWS services, add a separate `taskRoleArn` scoped
to just those permissions — never widen the execution role instead.

## Run it as a service

```bash
aws ecs create-service \
  --cluster training-cluster \
  --service-name training-app-svc \
  --task-definition training-app:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-0aaa1111", "subnet-0bbb2222"],
      "securityGroups": ["sg-0123456789abcdef0"],
      "assignPublicIp": "ENABLED"
    }
  }'
```

`assignPublicIp: ENABLED` is only needed if the subnets are public and the
task must reach the internet directly (e.g. to pull images) without a NAT
gateway. In a private subnet, route outbound traffic through a NAT gateway
or use VPC endpoints for ECR/S3 instead.

## Check status and logs

```bash
aws ecs describe-services \
  --cluster training-cluster --services training-app-svc \
  --query "services[0].[status,runningCount,desiredCount]"
# ["ACTIVE", 2, 2]

aws ecs list-tasks --cluster training-cluster --service-name training-app-svc

aws logs tail /ecs/training-app --since 10m
```

## Deploying a new version

```bash
# Build, tag, push a new image (as above), then register a new revision
aws ecs register-task-definition --cli-input-json file://task-def.json
# revision: 2

aws ecs update-service \
  --cluster training-cluster \
  --service training-app-svc \
  --task-definition training-app:2
```

ECS performs a **rolling deployment** by default: it starts new tasks on
revision 2, waits for them to pass health checks, then drains and stops
old tasks — no manual blue/green orchestration needed for a simple update.

!!! warning "Fargate is billed per vCPU-second and per GB-second, not per container"
    A task with `cpu: 256` (0.25 vCPU) and `memory: 512` (0.5 GB) is billed
    for exactly that much reserved capacity for as long as the task runs —
    whether or not it's busy. Two tasks running 24/7 cost roughly the same
    as one `t3.small`-class EC2 instance running 24/7, so right-size `cpu`/
    `memory` rather than over-provisioning "to be safe."

## ECS vs. Elastic Beanstalk vs. raw EC2

| | ECS/Fargate | Elastic Beanstalk (module 6) | Raw EC2 |
|---|---|---|---|
| Unit of deployment | Container image | Zip/WAR + platform | AMI / user-data script |
| Server management | None (Fargate) | Beanstalk manages it | You manage it |
| Best for | Microservices, polyglot stacks | Simple app + managed infra with less config | Full control, legacy workloads |

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws ecr create-repository --repository-name NAME` | Create a private image registry. |
| `aws ecr get-login-password \| docker login ...` | Authenticate Docker to ECR. |
| `aws ecs create-cluster --cluster-name NAME` | Create a cluster namespace. |
| `aws ecs register-task-definition --cli-input-json file://F` | Register a task definition revision. |
| `aws ecs create-service --launch-type FARGATE --network-configuration ...` | Run a task definition as a self-healing service. |
| `aws ecs update-service --task-definition FAMILY:REV` | Roll out a new task definition revision. |
| `aws ecs describe-services --cluster C --services S` | Check running vs. desired task count. |
| `aws logs tail /ecs/GROUP --since 10m` | Tail recent container logs. |

## How It Actually Works

ECS is fundamentally a **scheduler** — a control loop, not a container
runtime. The ECS control plane maintains the desired state you declare (a
service wanting N running tasks of a given task definition) and continuously
reconciles it against observed reality reported by agents running on your
compute, launching or stopping tasks to close the gap — the same
reconcile-loop pattern Kubernetes uses, just AWS-native and much simpler in
scope.

On **EC2 launch type**, the reconciliation is done by the **ECS Container
Agent**, a process running on each cluster instance that registers the
host's available CPU/memory with the control plane and receives task
placement instructions back; the agent then talks to the local Docker daemon
to actually start containers. On **Fargate launch type**, there's no
visible host at all — AWS provisions a Firecracker MicroVM per task
on-demand behind the scenes, meaning Fargate tasks get the same hardware
isolation boundary as Lambda functions, at the cost of a startup delay
(pulling the image and booting the MicroVM) that EC2-backed tasks avoid once
a host already has the image cached.

Task placement on EC2 launch type is a bin-packing problem the scheduler
solves per your chosen strategy (`binpack`, `spread`, or `random`):
`binpack` deliberately concentrates tasks onto the fewest possible instances
to maximize utilization (and let auto scaling shrink the cluster), while
`spread` distributes across AZs or instances for resilience — the scheduler
is optimizing a real constraint-satisfaction problem against each instance's
registered remaining CPU/memory headroom, not just round-robining.

## Exercise

1. Build a small HTTP app (any language) into a Docker image, push it to a
   new ECR repository.
2. Write a Fargate task definition with a separate execution role
   (`AmazonECSTaskExecutionRolePolicy`) and `awslogs` logging configured.
3. Create a cluster and a service with `desired-count: 2`, in two subnets
   across different AZs for resilience.
4. Confirm both tasks reach `RUNNING` and pass health checks via
   `describe-services`, then tail their combined logs.
5. Push a trivial code change, register a new task definition revision,
   and update the service — watch `describe-services` show the rolling
   swap from revision 1 to revision 2 with `desiredCount` never dropping
   below 2.
6. Delete the service (`update-service --desired-count 0` then
   `delete-service`) and the cluster when done so Fargate stops billing.
