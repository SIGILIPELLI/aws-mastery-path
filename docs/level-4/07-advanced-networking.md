# Advanced Networking (Direct Connect, Global Accelerator)

Level 3 module 1 connected networks over the public internet via VPN.
This module covers two services for when the internet path isn't good
enough: **Direct Connect** for a dedicated, private link to AWS, and
**Global Accelerator** for routing global end-user traffic onto AWS's
backbone as early as possible.

## Direct Connect: dedicated private connectivity

A VPN tunnel still traverses the public internet — variable latency,
shared bandwidth. **Direct Connect** is a physical, dedicated network
connection from your data center (via an AWS Direct Connect location)
to AWS, bypassing the public internet entirely.

```bash
aws directconnect create-connection \
  --location "EqDC2" \
  --bandwidth "1Gbps" \
  --connection-name "training-dc-primary"
# {
#     "connectionId": "dxcon-abc123de",
#     "connectionState": "requested"
# }
```

Provisioning is physical — after `create-connection`, AWS emails a
Letter of Authorization (LOA-CFA) that your network provider uses to
physically cross-connect at the Direct Connect location. This step
takes days to weeks, unlike every other resource in this path.

Once physically connected, create a **virtual interface (VIF)** to
actually route traffic:

```bash
aws directconnect create-private-virtual-interface \
  --connection-id dxcon-abc123de \
  --new-private-virtual-interface '{
    "virtualInterfaceName": "prod-vif",
    "vlan": 101,
    "asn": 65000,
    "amazonAddress": "192.168.1.1/30",
    "customerAddress": "192.168.1.2/30",
    "virtualGatewayId": "vgw-0abc123"
  }'
```

A **private VIF** connects to a VPC (via a virtual private gateway or
Transit Gateway); a **public VIF** reaches AWS public service endpoints
(S3, DynamoDB) over the dedicated link without transiting the internet
at all; a **transit VIF** connects to a Direct Connect gateway fanning
out to multiple Transit Gateways across regions.

## Direct Connect resiliency

A single Direct Connect connection is a single point of failure. AWS's
recommended resiliency models range from one connection at one
location (least resilient) to separate connections at two different
Direct Connect locations with diverse providers (highest resiliency).
Production setups keep a Site-to-Site VPN as backup, failing over
automatically via BGP route priority if the dedicated link drops.

## Global Accelerator: optimizing the last mile

Direct Connect solves data-center-to-AWS connectivity. **Global
Accelerator** solves a different problem: end users worldwide hitting a
public endpoint. It anycasts two static IPs from AWS's edge network, so
user traffic enters AWS's backbone at the nearest edge location instead
of traversing the public internet all the way to your origin region.

```bash
aws globalaccelerator create-accelerator \
  --name training-app-accelerator \
  --ip-address-type IPV4

aws globalaccelerator create-listener \
  --accelerator-arn arn:aws:globalaccelerator::123456789012:accelerator/abc123 \
  --port-ranges FromPort=443,ToPort=443 \
  --protocol TCP

aws globalaccelerator create-endpoint-group \
  --listener-arn arn:aws:globalaccelerator::123456789012:accelerator/abc123/listener/def456 \
  --endpoint-group-region us-east-1 \
  --endpoint-configurations EndpointId=arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/training-alb/ghi789,Weight=100 \
  --health-check-path /health
```

Adding a second endpoint group in another region gives Global
Accelerator automatic failover — if the primary region's health checks
fail, traffic shifts to the healthy region within seconds, faster than
DNS-based failover (Level 3 module 7) since it's not waiting on DNS TTL
expiry across the internet.

## Direct Connect vs. Global Accelerator vs. CloudFront

| | Solves | Direction |
|---|---|---|
| Direct Connect | Private link from your network to AWS | Data-center ↔ AWS |
| Global Accelerator | Fast, resilient entry point for TCP/UDP traffic (any protocol) | Internet users → AWS compute |
| CloudFront | Caching and edge delivery of HTTP(S) content | Internet users → cached content |

CloudFront caches content at the edge; Global Accelerator doesn't
cache anything — it just optimizes the network path to your actual
origin, which is why it works for non-HTTP protocols (gaming, VoIP)
that CloudFront can't handle.

## Gotchas

- **Direct Connect provisioning timelines are physical-world, not
  API-instant** — plan weeks of lead time; it cannot be spun up for a
  short-term project the way every other service in this path can.
- **A public VIF advertises your on-prem CIDR to AWS's public
  services** — misconfigured public VIF routing can expose more of
  your network to AWS's public endpoint space than intended; review
  BGP-advertised prefixes carefully.
- **Global Accelerator's static IPs don't change even during
  failover** — that's the point, but it also means DNS/firewall
  allowlists referencing those IPs don't need updates during an
  incident, unlike Route 53 failover which changes what a DNS lookup
  returns.
- **Global Accelerator health checks operate at the endpoint-group
  level** — a single unhealthy target within a healthy ALB doesn't
  trigger regional failover; only a failing health check on the
  accelerator's own configured check path/port does.
- **Direct Connect bandwidth is fixed at the port speed you
  provisioned** (1/10/100 Gbps) — traffic bursts beyond it queue or
  drop rather than auto-scaling like a VPN over the internet would
  (which has no such hard cap, just variable throughput).

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws directconnect create-connection` | Request a physical DX port |
| `aws directconnect create-private-virtual-interface` | Route DX traffic to a VPC |
| `aws directconnect create-public-virtual-interface` | Route DX traffic to AWS public endpoints |
| `aws globalaccelerator create-accelerator` | Create a global static-IP entry point |
| `aws globalaccelerator create-endpoint-group` | Add a regional target with health checks |

## How It Actually Works

**Direct Connect** bypasses the public internet entirely by provisioning a
dedicated physical (or hosted, sub-rate) fiber connection from your
router/colocation facility into an AWS Direct Connect location, where it
terminates onto AWS's own network hardware — from that point on, your
traffic to a VPC travels over AWS's private backbone rather than transiting
any public internet exchange point, which is the actual mechanism behind
Direct Connect's more predictable latency and jitter compared to a VPN over
the public internet (a VPN's tunnel still rides over the unpredictable,
shared public internet path underneath the encryption).

A **Direct Connect Gateway** exists because a raw DX connection alone can
only reach VPCs in the *same region* as the DX location — the gateway is a
global routing construct that lets one physical DX connection's BGP
sessions be associated with Virtual Private Gateways or Transit Gateways
across *multiple* regions, effectively projecting one physical link's reach
across your whole global VPC footprint via AWS's internal backbone rather
than requiring a separate physical connection per region.

**Global Accelerator** works through Anycast IP addressing: it advertises
the same static IP addresses from every AWS edge location worldwide
simultaneously, and relies on internet BGP routing itself to send a given
client's traffic to whichever advertising edge location is topologically
nearest — once traffic lands at that edge, it travels the rest of the way to
your actual resources over AWS's private backbone rather than the public
internet, similar in spirit to CloudFront but optimized for non-HTTP TCP/UDP
traffic and for a static IP that doesn't change even as you add or remove
backend endpoints across regions.

## Exercise

Design (no need to provision) a resilient hybrid network for a company
with one data center: two Direct Connect connections at different
locations for the primary path, a Site-to-Site VPN as backup, and a
Global Accelerator in front of a two-region ALB setup for their public
API. Write out which BGP/health-check settings would need to agree for
automatic failover to work end-to-end.
