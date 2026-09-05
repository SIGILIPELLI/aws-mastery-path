# Kubernetes on AWS (EKS)

ECS and Fargate (Level 2) are AWS's opinionated container schedulers.
**EKS** (Elastic Kubernetes Service) is AWS's managed control plane for
upstream Kubernetes — you get the full Kubernetes API, and portability
to any other Kubernetes environment, at the cost of more moving parts.

## EKS vs. ECS

| | ECS/Fargate | EKS |
|---|---|---|
| Control plane | Fully hidden | Managed, but you interact with the K8s API |
| Config language | Task definitions (JSON) | Kubernetes manifests (YAML) |
| Portability | AWS-only | Runs anywhere Kubernetes runs |
| Ecosystem | AWS-native integrations | Huge open-source ecosystem (Helm, operators) |
| Learning curve | Lower | Higher |

Choose EKS when you need multi-cloud portability, existing Kubernetes
expertise, or ecosystem tools (service meshes, operators) that assume
Kubernetes APIs.

## Create a cluster

```bash
aws eks create-cluster \
  --name training-cluster \
  --role-arn arn:aws:iam::123456789012:role/eksClusterRole \
  --resources-vpc-config subnetIds=subnet-0aaa111,subnet-0aaa222,securityGroupIds=sg-0aaa333

aws eks wait cluster-active --name training-cluster
```

The control plane takes 10-15 minutes to provision and bills a flat
hourly rate per cluster regardless of size — on top of whatever EC2 or
Fargate compute you attach for worker nodes.

Point `kubectl` at the new cluster:

```bash
aws eks update-kubeconfig --name training-cluster --region us-east-1
kubectl get svc
# NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
# kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   2m
```

`update-kubeconfig` writes an entry that shells out to `aws eks
get-token` for authentication — no long-lived credentials are stored in
`~/.kube/config`, but it does mean `kubectl` fails if your AWS CLI
credentials expire.

## Add worker nodes (managed node group)

```bash
aws eks create-nodegroup \
  --cluster-name training-cluster \
  --nodegroup-name standard-workers \
  --node-role arn:aws:iam::123456789012:role/eksNodeRole \
  --subnets subnet-0aaa111 subnet-0aaa222 \
  --scaling-config minSize=2,maxSize=4,desiredSize=2 \
  --instance-types t3.medium
```

A managed node group handles AMI selection, patching, and draining for
you. For serverless pods without managing EC2 at all, use **Fargate
profiles** instead — same underlying tech as ECS Fargate, but
scheduling Kubernetes pods:

```bash
aws eks create-fargate-profile \
  --cluster-name training-cluster \
  --fargate-profile-name default \
  --pod-execution-role-arn arn:aws:iam::123456789012:role/eksFargatePodRole \
  --selectors namespace=default
```

## Deploy a workload

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: training-app
spec:
  replicas: 3
  selector:
    matchLabels: { app: training-app }
  template:
    metadata:
      labels: { app: training-app }
    spec:
      containers:
        - name: training-app
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/training-app:latest
          ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata:
  name: training-app
spec:
  type: LoadBalancer
  selector: { app: training-app }
  ports: [{ port: 80, targetPort: 8080 }]
```

```bash
kubectl apply -f deployment.yaml
kubectl get pods
# NAME                            READY   STATUS    RESTARTS   AGE
# training-app-6f8d9c9b7d-2x4jp   1/1     Running   0          40s
```

A `Service` of type `LoadBalancer` triggers the AWS Load Balancer
Controller (installed separately as an add-on) to provision a real
Network or Application Load Balancer pointed at the pods.

## Gotchas

- **`aws-auth` ConfigMap / access entries** control who can talk to the
  cluster — IAM permissions to call `eks:*` APIs are separate from
  Kubernetes RBAC permissions inside the cluster. Creating a cluster
  doesn't automatically give teammates `kubectl` access.
- **Node groups need their own IAM role** with the
  `AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy`, and
  `AmazonEC2ContainerRegistryReadOnly` managed policies attached, or
  nodes never join the cluster.
- **Fargate profiles only match pods by namespace/labels at
  scheduling time** — a pod that doesn't match any profile's selector
  fails to schedule with no obvious error in `kubectl get pods`; check
  `kubectl describe pod` events.
- **Cluster and node group versions can skew** — EKS allows nodes up to
  two minor versions behind the control plane, which is useful during
  upgrades but easy to forget about and let drift too far.
- **Deleting a cluster leaves the VPC, load balancers, and EBS volumes
  behind** — `aws eks delete-cluster` does not clean up resources
  created by in-cluster controllers (like LoadBalancer Services);
  delete Kubernetes resources first or you'll orphan billable infra.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws eks create-cluster` | Provision the control plane |
| `aws eks update-kubeconfig` | Configure local `kubectl` |
| `aws eks create-nodegroup` | Add EC2-backed workers |
| `aws eks create-fargate-profile` | Add serverless pod scheduling |
| `kubectl get nodes` | List workers the control plane sees |
| `kubectl apply -f <file>` | Create/update resources from a manifest |
| `kubectl logs <pod>` | Tail container logs |

## How It Actually Works

EKS runs the upstream, unmodified **Kubernetes control plane**
(API server, etcd, scheduler, controller-manager) as a managed, highly
available service spread across multiple AZs — AWS operates this control
plane in an AWS-owned account/VPC, exposing only the API server endpoint to
you, which is why you never see or patch etcd directly, but also why the
control plane behaves exactly like any other Kubernetes cluster's for
tooling compatibility (`kubectl`, Helm, and the whole CNCF ecosystem work
unmodified).

Networking is where EKS diverges most from a typical on-prem Kubernetes
install: by default it uses the **AWS VPC CNI**, which assigns each pod a
*real, routable VPC IP address* drawn from your subnet's available range —
not an overlay-network IP like Flannel or Calico would typically assign.
The CNI plugin does this by pre-allocating extra elastic network interfaces
(and their secondary IPs) onto each worker node, ahead of pod scheduling, so
that when a new pod is scheduled, an IP is already available to attach
instantly rather than needing to provision one on the spot — this is also
the real cause of the well-known "IP exhaustion" failure mode: a node can
run out of assignable IPs (limited by both instance type and subnet size)
well before it runs out of CPU/memory, causing pods to get stuck in
`ContainerCreating`.

Node scheduling itself is genuine Kubernetes scheduling — the API server's
scheduler evaluates pod resource requests, taints/tolerations, and affinity
rules against each node's reported capacity, and only then binds a pod to a
node; `kubelet` on that node is what actually pulls the container image and
starts it via the container runtime. EKS's managed node groups just
automate the EC2 (or Fargate) provisioning side and register/deregister
nodes with this scheduler via a standard Auto Scaling Group lifecycle hook.

## Exercise

Create an EKS cluster with one managed node group of two `t3.medium`
nodes. Deploy a two-replica `Deployment` running any public image (e.g.
`nginx`), expose it with a `ClusterIP` service, and use `kubectl
port-forward` to reach it locally without provisioning a load balancer.
