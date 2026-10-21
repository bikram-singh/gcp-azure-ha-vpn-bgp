# Connecting GCP and Azure with an Active-Active HA VPN and BGP: A Step-by-Step Guide

**Subtitle:** Two IPsec tunnels, two BGP sessions, zero static routes. How I built a highly available multi-cloud VPN between Google Cloud and Azure, and proved it with a private-IP ping.

**Suggested tags:** GCP, Azure, Multicloud, VPN, BGP

> Image note for Medium: every image below links to the screenshot in the GitHub repo. In Medium's editor, paste the image URL on its own line and press Enter, or download the file from `docs/snapshots/` and upload it.
> Image base URL: `https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/`

---

Multicloud is no longer a buzzword. Sooner or later you will have a workload in Google Cloud that needs to talk privately to something in Azure: a database, a legacy app, a data platform. The fastest, cheapest way to give them private connectivity is a **site-to-site IPsec VPN**.

A single tunnel works, but it is a single point of failure. In this article I build the **highly available** version: **GCP HA VPN** on one side and an **Azure VPN Gateway in active-active mode** on the other, with **BGP** doing the routing. Two tunnels, two BGP sessions, and if one tunnel dies, traffic moves to the other automatically.

Everything is done from the cloud consoles, and I validate it at the end with a simple ping between two VMs using their **private IPs**.

📦 **Full guide, with all 65 screenshots:** https://github.com/bikram-singh/gcp-azure-ha-vpn-bgp

---

## What we are building

![Architecture](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/00-architecture/gcp-azure-ha-vpn-architecture.png)

- **GCP:** VPC `gcp-azure-vpc` (`10.10.0.0/20`), `gcp-vm` at `10.10.0.2`, HA VPN gateway, Cloud Router (ASN **65001**)
- **Azure:** VNet `gcp_azure_net` (`192.168.0.0/16`), `azure-vm` at `192.168.0.4`, VPN gateway in active-active mode (ASN **65002**)
- **Two tunnels:** GCP interface 0 ↔ Azure public IP 1, and GCP interface 1 ↔ Azure public IP 2
- **Two BGP sessions** over APIPA link-local addresses (`169.254.21.x` and `169.254.22.x`)

## The theory in five minutes

**HA VPN.** Google's HA VPN gateway has two interfaces, each with its own public IP. If you connect both to a peer, you get Google's 99.99% SLA. It supports only dynamic routing, so BGP is mandatory.

**Active-active on Azure.** A normal Azure VPN gateway is one active instance plus one standby. In active-active mode, *both* instances are live and each has its own public IP. That maps one-to-one to the two HA VPN interfaces, so tunnel 1 goes to Azure instance 1 and tunnel 2 to instance 2.

**Local Network Gateway (LNG).** This is Azure's way of describing the *other* side. Because GCP has two interfaces, you need **two LNGs**, each with the GCP public IP, ASN and BGP peer IP.

**BGP and APIPA.** BGP removes the need for static routes, and a dead tunnel's routes are withdrawn on their own. The BGP peers use link-local APIPA addresses. Azure only accepts `169.254.21.0` to `169.254.22.255`, so I used `169.254.21.x` for tunnel 1 and `169.254.22.x` for tunnel 2.

**Different ASNs.** It is eBGP, so each side needs its own ASN: `65001` for GCP and `65002` for Azure.

The addressing plan, which is the part I would write down *before* clicking anything:

| | Tunnel 1 | Tunnel 2 |
|---|---|---|
| GCP HA VPN interface | `34.152.79.70` | `34.177.46.16` |
| Azure gateway public IP | `4.156.244.86` | `135.234.206.228` |
| GCP (Cloud Router) BGP IP | `169.254.21.1` | `169.254.22.1` |
| Azure BGP IP | `169.254.21.2` | `169.254.22.2` |

One golden rule: **the private ranges on both sides must not overlap.** Here that is `10.10.0.0/20` vs `192.168.0.0/16`.

---

## Step 1: Build the Azure foundation

Create a resource group (`gcp_azure`, East US), then a virtual network `gcp_azure_net` with address space `192.168.0.0/16` and a workload subnet `192.168.0.0/24`.

![VNet IP addresses](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/02-azure-vnet/05-ip-addresses-subnet.png)

The VPN gateway needs its own dedicated subnet that **must be named `GatewaySubnet`**. I created a `/27` (`192.168.2.0/27`) by choosing *Virtual Network Gateway* as the subnet purpose.

![Gateway subnet](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/02-azure-vnet/07-add-gateway-subnet.png)

Then create a small Ubuntu VM (`azure-vm`) in the workload subnet. It got the private IP **`192.168.0.4`**.

![Azure VM private IP](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/04-gcp-vpc-and-vm/03-azure-vm-private-ip.png)

## Step 2: Build the GCP foundation

On the GCP side, create a custom-mode VPC `gcp-azure-vpc` with subnet `subnet-east-us` (`us-east1`, `10.10.0.0/20`) and a VM `gcp-vm`, which got **`10.10.0.2`**. Don't forget a firewall rule allowing ICMP and SSH from `192.168.0.0/16`.

![GCP VPC](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/04-gcp-vpc-and-vm/01-create-vpc.png)

## Step 3: Prove there is no connectivity yet

Before building anything, I ping each VM from the other using private IPs. Both fail. That is our "before" picture.

![Ping fails both ways](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/05-ping-before-vpn/01-ping-fails-both-ways.png)

## Step 4: Create two public IPs in Azure

Because I'm using active-active, I need **two Standard public IPs** on the Azure gateway. I create them *first* so I can hand them to GCP: `4.156.244.86` and `135.234.206.228`.

![Azure public IP](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/06-azure-public-ips/03-ip1-address.png)

## Step 5: Create the HA VPN gateway, Cloud Router and tunnels in GCP

In **Network Connectivity → VPN → Create VPN connection**, choose **High-availability (HA) VPN**.

1. Create the gateway `gcp-to-azure-vpn` in `gcp-azure-vpc`, `us-east1`. Google assigns two public IPs: `34.152.79.70` and `34.177.46.16`.

![Create HA VPN gateway](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/07-gcp-ha-vpn/03-create-ha-vpn-gateway.png)

2. Create a **peer VPN gateway** with **two interfaces**, and enter the two Azure public IPs.

![Peer gateway](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/07-gcp-ha-vpn/04-peer-vpn-gateway.png)

3. Create a **Cloud Router** `gcp-to-azure-vpn-router` with ASN **65001**.

![Cloud Router](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/07-gcp-ha-vpn/05-create-cloud-router.png)

4. Create the two tunnels with **IKEv2** and the same pre-shared key: interface 0 → Azure IP 1, interface 1 → Azure IP 2.

![Tunnel 1](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/07-gcp-ha-vpn/06-tunnel-1.png)

5. When GCP asks about BGP sessions, choose **Configure BGP sessions later**. We don't know Azure's APIPA addresses and ASN yet. The tunnels sit in *First handshake* for now. That is expected.

![First handshake](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/07-gcp-ha-vpn/09-tunnels-first-handshake.png)

> 🔐 Use a long random pre-shared key (`openssl rand -base64 32`), not something guessable. It is the only secret protecting the tunnel.

## Step 6: Create the active-active VPN gateway in Azure

Now the Azure side. Create a virtual network gateway:

- Type **VPN**, SKU **VpnGw2AZ**, **Generation 2**, in `gcp_azure_net`
- Public IP: **use existing** → the first IP
- **Enable active-active mode** → pick the second public IP
- **Configure BGP: Enabled**, ASN **65002**
- **Custom Azure APIPA BGP IP addresses:** `169.254.21.2` and `169.254.22.2`

![Active-active and BGP](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/08-azure-vnet-gateway/03-active-active-bgp-apipa.png)

The review page confirms everything:

![Review](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/08-azure-vnet-gateway/04-review.png)

Then go and get a coffee. A VPN gateway takes a while to deploy, often **30–45 minutes**.

## Step 7: Create two local network gateways

Each LNG describes one GCP interface:

| | LNG 1 | LNG 2 |
|---|---|---|
| Name | `azure-t0-gcp-network-gateway-1` | `azure-t0-gcp-network-gateway-2` |
| IP address (GCP interface) | `34.152.79.70` | `34.177.46.16` |
| Address space | `169.254.21.1/32` | `169.254.22.1/32` |
| BGP | Yes, ASN `65001` | Yes, ASN `65001` |
| BGP peer IP | `169.254.21.1` | `169.254.22.1` |

The ASN is the same on both because both interfaces belong to the same Cloud Router.

![LNG 1 basics](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/09-azure-local-network-gateways/02-lng1-basics.png)
![LNG 1 BGP](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/09-azure-local-network-gateways/04-lng1-advanced-asn-peer.png)

## Step 8: Create the two connections

On the gateway, go to **Settings → Connections → Add**, and create one connection per LNG:

- Type **Site-to-site (IPsec)**
- Shared key: **the same PSK you used in GCP**
- IKE protocol: **IKEv2**
- **Enable BGP: ✔**

`azure-to-gcp-tunnel-1` uses LNG 1, and `azure-to-gcp-tunnel-2` uses LNG 2.

![Connection settings](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/10-azure-connections/04-conn1-settings.png)

## Step 9: Bring up the BGP sessions in GCP

Back in GCP, open each tunnel and click **Configure BGP session**. Choose **Manually** for the IP allocation and enter the APIPA pair:

| | Tunnel 1 | Tunnel 2 |
|---|---|---|
| Peer ASN | `65002` | `65002` |
| Cloud Router BGP IP | `169.254.21.1` | `169.254.22.1` |
| BGP peer IP | `169.254.21.2` | `169.254.22.2` |

![BGP session](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/11-gcp-bgp-sessions/03-bgp-session-1-manual-ips.png)

After a minute, both tunnels flip to **Established**, and both BGP sessions show **BGP established**:

![Both established](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/11-gcp-bgp-sessions/06-both-established.png)

## The moment of truth

From `gcp-vm`, ping the Azure VM's **private IP**:

```
$ ping 192.168.0.4
64 bytes from 192.168.0.4: icmp_seq=1 ttl=63 time=16.3 ms
64 bytes from 192.168.0.4: icmp_seq=2 ttl=63 time=14.9 ms
64 bytes from 192.168.0.4: icmp_seq=3 ttl=63 time=15.1 ms
3 packets transmitted, 3 received, 0% packet loss
```

![GCP to Azure ping](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/12-validation/01-ping-gcp-to-azure-success.png)

Private connectivity between two clouds, about 15 ms round trip, no static routes. The earlier failed ping now succeeds.

## Things that tripped me up (or will trip you)

- **Order matters.** Create the Azure public IPs first, then GCP, then the Azure gateway, LNGs, and connections. Only then configure BGP on GCP. Otherwise you are guessing at IPs.
- **APIPA range.** Azure only takes `169.254.21.0–169.254.22.255`. Anything else will be rejected.
- **Matching values.** The LNG's BGP peer IP must equal the Cloud Router IP in GCP, and the GCP peer IP must equal the Azure APIPA IP. One typo and BGP never establishes.
- **BGP on both sides.** Enable BGP on the Azure gateway *and* on each connection.
- **Naming.** I named my gateway `gcp_to_azure_vpn`, which is confusing when the Azure side is the one that owns it. Name things by direction (`azure_to_gcp_vpn`) from the start.
- **Cost.** `VpnGw2AZ` and public IPs are billed hourly. Delete the resource group when you are done.

## Wrap-up

You now have a resilient GCP ↔ Azure link: two IPsec/IKEv2 tunnels, two BGP sessions, and an active-active gateway so a failure in either path does not take your connectivity with it.

From here you could:

- Add **Terraform** to build this repeatably (the console is great for learning, not for production)
- Add **custom route advertisements** on the Cloud Router to control which prefixes are shared
- Move to **Dedicated/Partner Interconnect + ExpressRoute** when you outgrow internet-based VPN bandwidth

The complete guide with all 65 screenshots is on GitHub: **https://github.com/bikram-singh/gcp-azure-ha-vpn-bgp**

If this helped, a clap or a follow is appreciated. I write regularly about GCP, DevOps and multicloud.

*Bikram Singh is a GCP Infrastructure & DevOps Lead Engineer.*
