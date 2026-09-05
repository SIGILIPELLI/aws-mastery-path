# Kubernetes at Scale (EKS Production Patterns)

Level 3 got a working EKS cluster with one static node group. Running
EKS in production means the cluster must scale automatically, deploy
without manual `kubectl apply`, and isolate multiple teams safely on
shared infrastructure.

## Cluster Autoscaler vs. Karpenter

| | Cluster Autoscaler | Karpenter |
|---|---|---|
| Scales | Existing node groups (ASGs) | Provisions nodes directly, no ASG |
| Instance selection | Fixed per node group | Dynamic, picks best-fit type per pending pod |
| Scale-down speed | Minutes | Faster, more aggressive consolidation |
| Maturity | Long-standing, Kubernetes SIG project | AWS-built, now the recommended default |

```yaml
# Karpenter NodePool: what kinds of nodes it's allowed to launch
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
  limits:
    cpu: 1000
```

```bash
kubectl apply -f nodepool.yaml
kubectl get nodeclaims  # Karpenter's record of nodes it has provisioned
```

Karpenter watches for unschedulable pods and provisions right-sized
nodes within seconds, rather than waiting on ASG scaling policies —
and it will mix spot and on-demand per the `capacity-type` requirement,
consolidating underutilized nodes automatically.

## Managed vs. self-managed vs. Fargate node groups

```bash
# Managed node group with a launch template for custom AMI/userdata
aws eks create-nodegroup \
  --cluster-name prod-cluster \
  --nodegroup-name gpu-workers \
  --node-role arn:aws:iam::123456789012:role/eksNodeRole \
  --subnets subnet-0aaa111 subnet-0aaa222 \
  --launch-template id=lt-0abc123def456,version=3 \
  --capacity-type SPOT
```

Managed node groups handle draining and rolling updates for you;
self-managed (a raw ASG you configure yourself) gives full control over
AMI and bootstrap but requires you to reimplement that lifecycle
management.

## GitOps deployment (Argo CD)

Rather than `kubectl apply` from a laptop, production clusters
reconcile continuously from a Git repo:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: training-app
spec:
  source:
    repoURL: https://github.com/org/training-app-manifests
    targetRevision: main
    path: overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

`selfHeal: true` means Argo CD reverts any manual `kubectl edit` drift
back to what's in Git — the cluster's actual state and Git become the
same thing, closing the loop CI/CD (Level 3 module 6) started.

## Multi-tenant namespace isolation

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    pods: "50"
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: team-a
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
    - from: [{ namespaceSelector: { matchLabels: { name: team-a } } }]
```

Namespaces alone don't enforce isolation — a `ResourceQuota` stops one
team from starving cluster capacity, and a default-deny `NetworkPolicy`
stops pods in one namespace from reaching pods in another (the CNI
plugin, e.g. the AWS VPC CNI or Calico, must support `NetworkPolicy`
enforcement — the default AWS VPC CNI needs this explicitly enabled).

## Gotchas

- **Karpenter needs its own IAM permissions to launch/terminate EC2
  instances directly** — broader than a managed node group's IAM
  needs, since it's acting as the ASG.
- **Spot interruptions give a 2-minute warning** — workloads on spot
  capacity need pod disruption budgets and graceful shutdown handling,
  or a 2-minute notice isn't enough for in-flight requests to drain.
- **`selfHeal` GitOps fights anyone who `kubectl edit`s in an
  emergency** — during an incident, either pause the Argo CD
  application first or your hotfix gets reverted within its next sync
  interval.
- **ResourceQuota without matching pod-level requests/limits does
  nothing** — a quota caps aggregate requested cpu/memory, but pods
  with no resource requests specified aren't accounted for correctly
  and can still oversubscribe nodes.
- **Multiple large node groups/NodePools increase EKS control plane
  API load** — very large clusters (1000+ nodes) need attention to API
  server request rate limits (`kubectl get --raw /metrics` for
  apiserver request duration).

## Cheat sheet

| Task | Tool/command |
|---|---|
| Auto-provision right-sized nodes | Karpenter `NodePool` |
| Scale existing ASG-backed node groups | Cluster Autoscaler |
| Continuous Git-driven deployment | Argo CD `Application` |
| Cap team resource usage | `ResourceQuota` |
| Isolate namespace traffic | `NetworkPolicy` |
| Spot capacity in managed node groups | `--capacity-type SPOT` |

## How It Actually Works

Running EKS at scale exposes control-plane and data-plane bottlenecks that
don't show up in a small cluster. The Kubernetes API server itself is the
central bottleneck any large cluster contends with: every `kubectl` command,
every controller reconcile loop, and every kubelet heartbeat all funnel
through the same API server's request-handling and etcd-backed storage —
EKS scales the control plane horizontally behind the scenes as load
increases, but etcd's fundamentally consensus-based (Raft) write path means
write-heavy workloads (a very high rate of pod churn, for instance) can
still produce API latency that no amount of worker-node scaling fixes,
because the bottleneck is in the control plane, not your nodes.

**Cluster Autoscaler** and **Karpenter** solve node-level scaling with
different mechanisms: Cluster Autoscaler works by watching for
`Unschedulable` pods (ones the Kubernetes scheduler already tried and
failed to place) and then scaling a specific, pre-defined Auto Scaling
Group up to fit them — it's reactive to existing ASG shapes. Karpenter
instead evaluates unschedulable pods' actual resource requests directly and
provisions **exactly-fitting** EC2 capacity on demand via the EC2 API,
without needing pre-defined ASGs or instance-type node groups at all —
this is a fundamentally different provisioning model (direct
just-in-time bin-packing vs. scaling a pre-shaped group) and is why
Karpenter typically achieves tighter bin-packing and faster scale-out.

At high pod density, the VPC CNI's IP-pre-allocation model (module 02 of
this level's prerequisite) becomes the dominant constraint: large clusters
routinely need either larger CIDR ranges, IPv6 (which sidesteps IPv4
exhaustion by giving nodes a vastly larger allocatable address space), or
an alternate CNI mode (prefix delegation, which allocates a whole `/28` IP
prefix to a network interface at once instead of individual IPs) to avoid
running out of assignable pod IPs before running out of actual node
capacity.

## Exercise

Install Karpenter on an EKS cluster, define a `NodePool` limited to
`m5` and `c5` spot instances with a 500 vCPU cap, and deploy a
`Deployment` requesting more CPU than any current node has free.
Confirm Karpenter provisions a new node within roughly a minute by
watching `kubectl get nodeclaims -w`.
