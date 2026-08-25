# Advanced VPC (Peering, Transit Gateway, VPN)

Level 1 gave you a single VPC with public/private subnets. Real
organizations run dozens of VPCs — one per team, environment, or
account — and need them to talk to each other and to on-premises
networks. This module covers the three main connectivity tools: VPC
peering, Transit Gateway, and Site-to-Site VPN.

## VPC peering

A peering connection links two VPCs so traffic routes directly between
them over AWS's backbone, without traversing the public internet.
Peering is **non-transitive**: if VPC A peers with B, and B peers with
C, A cannot reach C through B.

```bash
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-0111111111111aaaa \
  --peer-vpc-id vpc-0222222222222bbbb
# {
#     "VpcPeeringConnection": {
#         "Status": { "Code": "initiating-request" },
#         "VpcPeeringConnectionId": "pcx-0abc123def456"
#     }
# }

# The owner of the peer VPC must accept it
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-0abc123def456
```

Accepting the connection is not enough — each side needs a **route
table entry** pointing the other VPC's CIDR at the peering connection,
and security groups must allow the traffic:

```bash
aws ec2 create-route \
  --route-table-id rtb-0111111111111aaaa \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id pcx-0abc123def456
```

Non-overlapping CIDR ranges are mandatory — peering (and Transit
Gateway) cannot route between VPCs whose address ranges collide.

## Transit Gateway: hub-and-spoke at scale

Peering connections grow as O(n²) — 10 VPCs fully meshed need 45
connections. **Transit Gateway (TGW)** is a regional router that every
VPC attaches to once, turning the mesh into a hub-and-spoke.

```bash
aws ec2 create-transit-gateway \
  --description "prod-hub" \
  --options AmazonSideAsn=64512,DefaultRouteTableAssociation=enable

aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-0abc123def456789 \
  --vpc-id vpc-0111111111111aaaa \
  --subnet-ids subnet-0aaa111 subnet-0aaa222
```

TGW route tables are separate from VPC route tables — you still add a
route in each VPC's route table pointing relevant CIDRs at the TGW, and
a route in the TGW route table pointing back at each attachment. TGW
also supports **multicast** and cross-region peering between transit
gateways, which plain VPC peering doesn't.

Unlike VPC peering (free for the connection itself, you pay only for
data transfer), TGW bills an hourly charge **per attachment** plus a
per-GB processing fee — for two or three VPCs, peering is usually
cheaper; TGW wins once you're managing five or more.

## Site-to-Site VPN

Connects an on-premises network to a VPC over IPsec tunnels across the
public internet — no dedicated hardware required, unlike Direct
Connect (covered in Level 4).

```bash
# The customer gateway represents your on-prem router/firewall
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip 203.0.113.10 \
  --bgp-asn 65000

# The virtual private gateway is the AWS side, attached to your VPC
aws ec2 create-vpn-gateway --type ipsec.1
aws ec2 attach-vpn-gateway \
  --vpn-gateway-id vgw-0abc123 --vpc-id vpc-0111111111111aaaa

aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-0def456 \
  --vpn-gateway-id vgw-0abc123 \
  --options StaticRoutesOnly=false
```

Each VPN connection provisions **two tunnels** to different AWS
endpoints for redundancy — configure your on-prem device to use both,
or you lose failover if AWS performs maintenance on one endpoint. With
`StaticRoutesOnly=false`, routes propagate dynamically via BGP; static
routing requires you to add `--vpn-connection-route` entries manually
and doesn't auto-heal after topology changes.

You can attach a VPN connection to a Transit Gateway instead of a VPC
directly (`create-vpn-connection --transit-gateway-id ...`), letting
one VPN serve every VPC on the TGW.

## Gotchas

- **CIDR overlap** breaks peering and TGW routing silently at the
  route-table level — plan address ranges org-wide before creating
  VPCs, since re-IP'ing a live VPC is painful.
- **Route table propagation is manual for peering**, automatic (via
  TGW route tables) for Transit Gateway — a common outage cause is a
  peering connection that's "active" but has no route pointing to it.
- **Security groups can't be shared across peered VPCs by ID** unless
  the VPCs are in the same region; cross-region peering requires CIDR-
  based security group rules instead.
- **DNS resolution across peering** is off by default — you must
  explicitly enable `--modify-vpc-peering-connection-options` with
  `AllowDnsResolutionFromRemoteVpc` for private DNS names to resolve.
- **TGW route table associations** default to a single shared route
  table; production setups usually create separate route tables per
  environment (prod/dev) so a dev VPC attachment can't route into prod.

## Cheat sheet

| Tool | Topology | Transitive? | Typical use |
|---|---|---|---|
| VPC Peering | Point-to-point | No | 2-4 VPCs, simple mesh |
| Transit Gateway | Hub-and-spoke | Yes (via TGW route tables) | 5+ VPCs, multi-account |
| Site-to-Site VPN | On-prem ↔ AWS | N/A | Quick on-prem connectivity |
| Direct Connect (L4) | On-prem ↔ AWS | N/A | Dedicated, high-bandwidth |

## Exercise

Sketch (on paper or in a text file) a three-VPC topology — `prod`,
`staging`, and `shared-services` — connected through a single Transit
Gateway. Write the `aws ec2` commands you'd run to: create the TGW,
attach all three VPCs, and add the route-table entries so that `prod`
and `staging` can each reach `shared-services` but not each other.
