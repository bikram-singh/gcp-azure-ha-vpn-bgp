<!--
PUBLISHING NOTES (delete this block before you publish):
- Medium does not support markdown tables, so every table below is an image (docs/article/*.png in the repo).
- Every image URL points at the GitHub repo (branch main). Push the repo first, then in Medium's editor paste each image URL on its own line and press Enter, or download the file and upload it.
- Image base URL: https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/
- Medium supports H1/H2 headings, bold, bullet lists and code blocks. Use "##" lines as section headings.
-->

# Connecting GCP and Azure with an Active-Active HA VPN and BGP

**GCP · Azure · Multicloud · HA VPN · BGP · Cloud Router · VPN Gateway**

*A practical, screenshot-by-screenshot walkthrough of a highly available site-to-site VPN between Google Cloud and Azure: GCP HA VPN with Cloud Router on one side, an Azure VPN gateway in active-active mode on the other, two IPsec/IKEv2 tunnels, and two BGP sessions over APIPA addresses. Built from scratch and proved with a private-IP ping.*

![Architecture: GCP HA VPN to Azure active-active VPN gateway](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/00-architecture/gcp-azure-ha-vpn-architecture.png)

## ❓ Why HA VPN + BGP?

Sooner or later a workload in Google Cloud needs to talk privately to something in Azure: a database, a legacy app, a data platform. The fastest and cheapest way to give them private connectivity is a site-to-site IPsec VPN over the internet.

A single tunnel works, but it is a single point of failure: one gateway reboot, one maintenance window, one flapping path, and the link is gone. Static routes make it worse, because someone has to edit routes on both clouds every time a network changes.

The fix has two parts:

- **Two of everything.** GCP's HA VPN gateway has two interfaces. Azure's VPN gateway can run in active-active mode, with two instances that are both live. Connect them pairwise and you get two independent tunnels.
- **BGP instead of static routes.** Each side advertises its networks dynamically. When a tunnel dies, its routes are withdrawn and traffic moves to the surviving tunnel on its own.

I built this end to end across both clouds, using only the consoles, to see exactly how the pieces fit together, including the order you have to do them in. This article is that build, step by step.

## 🔑 Key Facts

![Key facts](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/article/01-key-facts.png)

## ✨ What Makes This Different

![What makes this different](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/article/02-what-makes-different.png)

## 💡 Why HA VPN + BGP: the benefits, mapped to what I built

![Benefits mapped to this build](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/article/03-why-ha-vpn-bgp.png)

## 🏛️ Architecture

```
            GCP (us-east1)                                  Azure (East US)
  ┌─────────────────────────────┐                 ┌──────────────────────────────────────┐
  │ VPC: gcp-azure-vpc          │                 │ VNet: gcp_azure_net  192.168.0.0/16  │
  │ 10.10.0.0/20                │                 │  ├─ gcp_azure_east-us 192.168.0.0/24 │
  │  └─ gcp-vm   10.10.0.2      │                 │  │    └─ azure-vm  192.168.0.4       │
  │                             │                 │  └─ GatewaySubnet 192.168.2.0/27     │
  │ HA VPN GW: gcp-to-azure-vpn │                 │                                      │
  │  if0 34.152.79.70 ──────────┼── Tunnel 1 ─────┼── 4.156.244.86     (VNG instance 1)  │
  │  if1 34.177.46.16 ──────────┼── Tunnel 2 ─────┼── 135.234.206.228  (VNG instance 2)  │
  │                             │   IPsec/IKEv2   │ VNG: active-active, VpnGw2AZ         │
  │ Cloud Router  ASN 65001     │◄──── BGP ──────►│ BGP ASN 65002                        │
  │  169.254.21.1 / 169.254.22.1│  (APIPA /30s)   │  169.254.21.2 / 169.254.22.2         │
  └─────────────────────────────┘                 └──────────────────────────────────────┘
```

Two tunnels, two BGP sessions, one private ping at the end. Here is the exact mapping I used:

![Tunnel and BGP mapping](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/article/05-tunnel-mapping.png)

## 🏗️ What Was Built

![What was built](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/article/06-what-was-built.png)

## 🧠 The concepts you need (five minutes of theory)

**Site-to-site IPsec VPN.** An encrypted tunnel over the internet between two networks. Each end needs a public IP, an agreed IKE version (I used **IKEv2**) and a pre-shared key.

**GCP HA VPN.** A regional Cloud VPN gateway with **two interfaces**, each with its own public IP. With two tunnels over the two interfaces you get Google's 99.99% availability design. HA VPN supports **dynamic routing (BGP) only**, so BGP is not optional.

**Cloud Router.** The BGP speaker on the GCP side. It learns Azure's prefixes and advertises the VPC subnets. It has its own ASN, `65001` in this build.

**Azure VPN gateway in active-active mode.** A normal Azure VPN gateway is one active instance plus one standby. In active-active mode **both instances are live**, each with its own public IP. That maps one-to-one onto the two HA VPN interfaces: tunnel 1 lands on Azure instance 1, tunnel 2 on instance 2.

**Local Network Gateway (LNG).** Azure's way of describing the *other* side: its public IP, its BGP ASN and its BGP peer IP. GCP has two interfaces, so you need **two LNGs**.

**BGP and APIPA.** The two BGP peers talk over link-local **APIPA** addresses (`169.254.x.x`). Azure only accepts `169.254.21.0` to `169.254.22.255` for this, which is why I used `169.254.21.x` for tunnel 1 and `169.254.22.x` for tunnel 2.

**ASNs.** It is eBGP, so each side needs a *different* private ASN: `65001` for GCP, `65002` for Azure.

And the plan I would write down *before* clicking anything:

![Design and addressing plan](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/article/04-addressing-plan.png)

One golden rule: **the private ranges on both sides must not overlap.** Here that is `10.10.0.0/20` against `192.168.0.0/16`.

## 🔄 The build order (this matters)

```
 Azure RG / VNet / VM ──► GCP VPC / VM ──► Baseline ping (fails) ──► Azure public IPs
        │
        ▼
 GCP HA VPN gateway + Cloud Router + 2 tunnels   (BGP "later")
        │
        ▼
 Azure VNG (active-active, BGP, APIPA) ──► 2 Local Network Gateways ──► 2 Connections
        │
        ▼
 GCP BGP sessions (manual APIPA) ──► Tunnels Established ──► Ping succeeds ✅
```

The order exists because of a chicken-and-egg problem: GCP needs Azure's public IPs, and Azure needs GCP's public IPs, ASN and BGP addresses. Creating the two Azure public IPs first breaks the loop.

## 🔹 Part 1: The Azure foundation

Everything on the Azure side lives in one resource group, `gcp_azure`, in East US.

Then the network: a VNet `gcp_azure_net` with address space `192.168.0.0/16` and a workload subnet `gcp_azure_east-us` on `192.168.0.0/24`.

![Azure VNet subnet](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/02-azure-vnet/05-ip-addresses-subnet.png)

The VPN gateway needs a subnet of its own, and **it must be named `GatewaySubnet`**. In the portal you get the name for free by choosing *Virtual Network Gateway* as the subnet purpose. I used a `/27` (`192.168.2.0/27`), which is the minimum Microsoft recommends.

![GatewaySubnet](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/02-azure-vnet/07-add-gateway-subnet.png)

Last, a small Ubuntu 24.04 VM (`azure-vm`, size B1s) in the workload subnet. It got the private IP **`192.168.0.4`**. That VM is our test target.

![Azure VM private IP](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/04-gcp-vpc-and-vm/03-azure-vm-private-ip.png)

## 🔹 Part 2: The GCP foundation, and a ping that must fail

On GCP I created a custom-mode VPC `gcp-azure-vpc` with one subnet, `subnet-east-us` in `us-east1` on `10.10.0.0/20`, and a VM `gcp-vm` that got **`10.10.0.2`**.

![GCP VPC](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/04-gcp-vpc-and-vm/01-create-vpc.png)

Before building any VPN I tested the "before" state: ping the Azure VM from the GCP VM and the GCP VM from Azure, using private IPs. Both hang with no replies, which is exactly what you would expect, because the two private networks have no path between them.

![Baseline ping fails](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/05-ping-before-vpn/01-ping-fails-both-ways.png)

Capturing the failure first makes the success at the end mean something.

## 🔹 Part 3: Two public IPs for the Azure gateway

An active-active Azure gateway needs **two** public IPs, both Standard SKU and regional. I created `gcp_azure_vpn_ip1` (`4.156.244.86`) and `gcp_azure_vpn_ip2` (`135.234.206.228`) up front, so I could hand them to GCP.

![Azure public IP](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/06-azure-public-ips/03-ip1-address.png)

## 🔹 Part 4: The GCP side: HA VPN gateway, Cloud Router and tunnels

In **Network Connectivity → VPN → Create VPN connection** you choose between Classic VPN and HA VPN. HA VPN is the one with dynamic routing and the availability SLA.

![Select HA VPN](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/07-gcp-ha-vpn/02-select-ha-vpn.png)

The sequence:

1. **Create the gateway** `gcp-to-azure-vpn` in `gcp-azure-vpc`, `us-east1`, IPv4. Google allocates two public IPs: `34.152.79.70` (interface 0) and `34.177.46.16` (interface 1).
2. **Create a peer VPN gateway** `gcp-to-azure-vpn-gateway` with **two interfaces**, entering the two Azure public IPs.
3. **Create a Cloud Router** `gcp-to-azure-vpn-router` with ASN **65001**.
4. **Create two tunnels** with IKEv2 and the same pre-shared key: interface 0 to Azure IP 1, interface 1 to Azure IP 2.
5. **Defer BGP.** At the BGP step I chose *Configure BGP sessions later*, because the Azure APIPA addresses and ASN did not exist yet.

![Peer VPN gateway with two interfaces](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/07-gcp-ha-vpn/04-peer-vpn-gateway.png)

![Cloud Router with ASN 65001](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/07-gcp-ha-vpn/05-create-cloud-router.png)

Equivalent CLI, for reference (I used the console for the build, so check the flags against `--help` before relying on them):

```bash
# HA VPN gateway: two interfaces, two public IPs allocated by Google
gcloud compute vpn-gateways create gcp-to-azure-vpn \
  --network=gcp-azure-vpc --region=us-east1

# Peer gateway: the two Azure public IPs
gcloud compute external-vpn-gateways create gcp-to-azure-vpn-gateway \
  --interfaces=0=4.156.244.86,1=135.234.206.228

# Cloud Router with Google's ASN
gcloud compute routers create gcp-to-azure-vpn-router \
  --network=gcp-azure-vpc --region=us-east1 --asn=65001

# Tunnel 1 (repeat with interface 1 / peer interface 1 for tunnel 2)
gcloud compute vpn-tunnels create gcp-to-azure-vpn-tunnel-1 \
  --region=us-east1 --vpn-gateway=gcp-to-azure-vpn --interface=0 \
  --peer-external-gateway=gcp-to-azure-vpn-gateway \
  --peer-external-gateway-interface=0 \
  --ike-version=2 --shared-secret="$PSK" \
  --router=gcp-to-azure-vpn-router
```

### 🔍 Key behaviors I found

**The tunnels cannot finish yet, and that is fine.** Right after creation both tunnels sit in **First handshake**, with BGP showing as not configured. Nothing is wrong: the Azure side does not exist, so there is nobody to complete the handshake with.

![Tunnels in First handshake](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/07-gcp-ha-vpn/09-tunnels-first-handshake.png)

**The pre-shared key is shown once.** GCP warns that the key can't be retrieved after the form is closed. Generate a long random key (`openssl rand -base64 32`), store it somewhere safe, and use the same value on both Azure connections.

## 🔹 Part 5: The Azure active-active VPN gateway

On the Azure side, create a **Virtual network gateway**:

- Type **VPN**, SKU **VpnGw2AZ**, **Generation 2**, in `gcp_azure_net` (it picks up `GatewaySubnet`)
- **Public IP address:** use existing, `gcp_azure_vpn_ip1`
- **Enable active-active mode:** on, with `gcp_azure_vpn_ip2` as the second public IP
- **Configure BGP:** enabled, ASN **65002**
- **Custom Azure APIPA BGP IP addresses:** `169.254.21.2` and `169.254.22.2`

![Active-active, BGP and APIPA](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/08-azure-vnet-gateway/03-active-active-bgp-apipa.png)

The review page confirms the settings before you commit:

![Gateway review](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/08-azure-vnet-gateway/04-review.png)

### 🔍 Key behaviors I found

**BGP needs the right SKU.** The connection page later reminds you: "To enable BGP, the SKU has to be Standard or higher". `VpnGw2AZ` clears that bar and gives zone redundancy as well.

**It takes a while.** VPN gateways are slow to provision, often 30 to 45 minutes. Plan the rest of the build around it.

**A detail on the APIPA fields.** On my review page both APIPA addresses appeared together under the first custom-APIPA field, and the second-instance field showed `None`. The tunnels and BGP sessions still came up. If you are designing this for production, check Microsoft's documentation on how APIPA addresses map to the two gateway instances.

## 🔹 Part 6: Two Local Network Gateways

Azure needs to know what is on the other end of each tunnel, so each GCP interface gets its own **Local Network Gateway**:

- **LNG 1** `azure-t0-gcp-network-gateway-1`: IP `34.152.79.70`, address space `169.254.21.1/32`, BGP on, ASN `65001`, BGP peer IP `169.254.21.1`
- **LNG 2** `azure-t0-gcp-network-gateway-2`: IP `34.177.46.16`, address space `169.254.22.1/32`, BGP on, ASN `65001`, BGP peer IP `169.254.22.1`

![LNG 1 basics](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/09-azure-local-network-gateways/02-lng1-basics.png)

![LNG 1 BGP settings](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/09-azure-local-network-gateways/04-lng1-advanced-asn-peer.png)

### 🔍 Key behaviors I found

**Both LNGs use the same ASN.** Both GCP interfaces belong to the same Cloud Router, so both LNGs carry Google's ASN, `65001`. What differs per LNG is the public IP and the BGP peer IP.

**The `/32` is the BGP peer address, not your VPC.** The address space on each LNG is just the Cloud Router's APIPA address (`169.254.21.1/32`). Your actual VPC ranges are learned over BGP, so you never type them in.

**These values must match what you configure on GCP later.** The LNG's BGP peer IP has to equal the Cloud Router IP you assign in Part 8, exactly.

## 🔹 Part 7: Two Azure connections

On the gateway, go to **Settings → Connections → Add**, and create one site-to-site (IPsec) connection per tunnel:

- `azure-to-gcp-tunnel-1` with LNG 1
- `azure-to-gcp-tunnel-2` with LNG 2
- Authentication: shared key (PSK), **identical** to the one on the GCP tunnels
- IKE protocol: **IKEv2**
- **Enable BGP:** checked

![Connection settings](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/10-azure-connections/04-conn1-settings.png)

### 🔍 Key behaviors I found

**BGP has to be on in two places.** It must be enabled on the gateway (Part 5) *and* on each connection. Missing either one is a classic reason for a tunnel that is up but a BGP session that never forms.

**Naming matters more than you think.** My own note on the connection screenshot says the gateway should have been named `azure-to-gcp-vpn`; I had created it as `gcp_to_azure_vpn`, which reads backwards when the gateway belongs to Azure. Name resources by direction from the start.

## 🔹 Part 8: Bringing up BGP on the GCP side

Back in GCP, each tunnel now needs its BGP session. For each one, **Configure BGP session**, and instead of letting Google pick addresses, choose **Allocate BGP IPv4 address: Manually**:

- **Tunnel 1:** peer ASN `65002`, Cloud Router IP `169.254.21.1`, BGP peer IP `169.254.21.2`
- **Tunnel 2:** peer ASN `65002`, Cloud Router IP `169.254.22.1`, BGP peer IP `169.254.22.2`

![Manual APIPA addresses](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/11-gcp-bgp-sessions/03-bgp-session-1-manual-ips.png)

Once the Azure side was complete, the first session showed as established:

![BGP session 1 established](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/11-gcp-bgp-sessions/04-bgp-session-1-established.png)

and once the second was configured, both tunnels showed **Established** with **BGP established**:

![Both tunnels and BGP established](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/11-gcp-bgp-sessions/06-both-established.png)

Every address on this screen has to line up with something you typed earlier: the Cloud Router IP with the LNG's BGP peer IP, and the BGP peer IP with the Azure APIPA address. That is why I keep calling it the part where one typo ruins your afternoon.

## 🎯 The moment of truth: a private ping across clouds

From `gcp-vm`, ping the Azure VM's **private IP**:

```
$ ping 192.168.0.4
64 bytes from 192.168.0.4: icmp_seq=1 ttl=63 time=16.3 ms
64 bytes from 192.168.0.4: icmp_seq=2 ttl=63 time=14.9 ms
64 bytes from 192.168.0.4: icmp_seq=3 ttl=63 time=15.1 ms
3 packets transmitted, 3 received, 0% packet loss
rtt min/avg/max/mdev = 14.893/15.435/16.291/0.612 ms
```

![GCP to Azure ping success](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/snapshots/12-validation/01-ping-gcp-to-azure-success.png)

Private connectivity between two clouds, about 15 ms round trip, no static routes. The same ping that hung in Part 2 now succeeds. Note the `ttl=63`: one hop was decremented on the way, which fits traffic crossing the VPN gateway.

Two follow-ups I would do next (I have only captured the GCP-to-Azure direction): ping `10.10.0.2` from `azure-vm` to confirm the reverse direction, and run a continuous ping while deleting one tunnel to watch BGP shift traffic to the other.

## 🧾 Cheat Sheet: Gotchas Worth Remembering

![Gotchas cheat sheet](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/article/07-gotchas.png)

## 🕵️ Observations from the Build

These came up during the actual build rather than from the docs:

![Observations from the build](https://raw.githubusercontent.com/bikram-singh/gcp-azure-ha-vpn-bgp/main/docs/article/08-observations.png)

## 💰 Cost and cleanup

A VPN gateway is not free to leave running. **`VpnGw2AZ` and the two Standard public IPs are billed hourly while they exist**, and both clouds charge for the tunnels and for data transfer. Check the official pricing pages before leaving a lab like this up.

To clean up:

1. **Azure:** delete the resource group `gcp_azure`. That removes the VNet, VM, gateway, LNGs, connections and public IPs in one go.
2. **GCP:** delete the two VPN tunnels, the HA VPN gateway, the peer VPN gateway, the Cloud Router, `gcp-vm`, and finally the VPC `gcp-azure-vpc`.

## 🔐 A note on secrets

The pre-shared key is the only thing protecting the tunnels, and screenshots of the tunnel and connection forms can show it. If you publish a build like this, use a throwaway key, destroy the lab afterwards or rotate the key, and blur anything sensitive (keys, subscription IDs, email addresses) in your screenshots.

## 📸 Snapshots

All 64 console screenshots from this build, sorted by step, are in the repository under `docs/snapshots/`.

## 🎯 Wrapping Up

The mental model is simple: two endpoints on each side, two tunnels, BGP on top. In practice the hard part is not any single screen but the **order of operations** and the **exact matching of values** across two consoles: public IPs, ASNs, APIPA addresses and the pre-shared key. Get those lined up on paper first, and the build itself is straightforward.

Natural next steps:

- **Terraform it.** The console is great for learning; production needs repeatability.
- **Control what you advertise.** Use custom route advertisements on the Cloud Router instead of advertising every subnet.
- **Test failover properly.** Take down one tunnel under a continuous ping and watch BGP converge.
- **Outgrow the internet.** When bandwidth or latency demands it, move to Interconnect and ExpressRoute.

If you are connecting GCP to Azure, give this a try in a throwaway project before it goes near production.

## 🔗 Repository

🌐 **gcp-azure-ha-vpn-bgp**: https://github.com/bikram-singh/gcp-azure-ha-vpn-bgp

*Bikram Singh is a GCP Infrastructure & DevOps Lead Engineer.*
