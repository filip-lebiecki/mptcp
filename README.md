# MPTCP Site-to-Site VPN with Bandwidth Aggregation

Aggregate **two routed links into one logical pipe** for a single TCP connection — with seamless failover — using **Multipath TCP (MPTCP)** in the mainline Linux kernel, then wrap it in a transparent, encrypted **site-to-site VPN** with [sing-box](https://sing-box.sagernet.org/).

This repo accompanies the video and contains the ready-to-use `sing-box` configs. Below are all the commands, step by step.

> Unlike LACP (Layer 2) or ECMP (per-flow Layer 3/4 hashing), MPTCP can split and aggregate a **single TCP flow** across multiple paths.

---

## Topology

```
          ┌─────────── link A: 10.0.0.0/24 (eth2) ───────────┐
 node1                                                         node2
 10.0.0.1 ●                                                  ● 10.0.0.2
 10.1.0.1 ●                                                  ● 10.1.0.2
          └─────────── link B: 10.1.0.0/24 (eth3) ───────────┘

 LAN behind node1: 172.16.0.0/24            LAN behind node2: 172.16.1.0/24
```

Goal: reachability (and aggregated bandwidth) between `172.16.0.0/24` and `172.16.1.0/24`.

> The two interconnect links use `10.x` private addressing for simplicity. In the real world they'd be separate WAN uplinks with public IPs — see [Both nodes behind NAT](#5-both-nodes-behind-nat-public-ips).

---

## Prerequisites

MPTCP ships in the mainline kernel (since 5.6, 2020) and is usually on by default. Make sure, on **both** nodes:

```bash
sysctl -w net.mptcp.enabled=1
```

Tools used in the demo: `iperf3`, `bmon`, `mptcpize` (ships with the `mptcpd` package), `sstui`/`ss`.

---

## 1. Baseline — static routes (single link only)

Plain static routes get you reachability, but a single TCP flow rides **one** link.

```bash
# node1
ip route add 172.16.1.0/24 via 10.0.0.2

# node2
ip route add 172.16.0.0/24 via 10.0.0.1
```

```bash
# test (node1) — ~1 Gbit, only eth2 used
iperf3 -c 172.16.1.1
```

LACP won't help (links are in different L3 subnets), and even if it could, a single flow is still pinned to one member.

---

## 2. ECMP — use both links (per-flow only)

Remove the static routes first:

```bash
# node1
ip route del 172.16.1.0/24 via 10.0.0.2
# node2
ip route del 172.16.0.0/24 via 10.0.0.1
```

Build a nexthop group and route through it:

```bash
# node1
ip nexthop add id 10 via 10.0.0.2 dev eth2
ip nexthop add id 11 via 10.1.0.2 dev eth3
ip nexthop add id 100 group 10/11
ip route add 172.16.1.0/24 nhid 100

# node2
ip nexthop add id 10 via 10.0.0.1 dev eth2
ip nexthop add id 11 via 10.1.0.1 dev eth3
ip nexthop add id 100 group 10/11
ip route add 172.16.0.0/24 nhid 100
```

By default the kernel hashes on **source/dest IP only**, so a single flow always picks the same path. Add Layer-4 (ports) to the hash, on **both** nodes:

```bash
sysctl -w net.ipv4.fib_multipath_hash_policy=1
```

Even then, a single connection has constant ports → constant hash → one link. ECMP only spreads **multiple** flows:

```bash
# node1 — 4 parallel streams now fill both links
iperf3 -c 172.16.1.1 -P 4
```

**Limitation:** one big single-stream transfer is still stuck on one link. That's what MPTCP fixes.

> Tip: enabling L4 hashing (`fib_multipath_hash_policy=1`) is generally good practice for ECMP on Linux.

To continue with MPTCP, tear down the ECMP routes/nexthops (or just reboot for a clean slate).

---

## 3. MPTCP — split a single flow (minimal demo)

Each node has two interfaces (`eth2`, `eth3`). We treat `eth2` (`10.0.0.x`) as the primary path the client connects to first.

**node2 (server)** — advertise the second address and raise limits:

```bash
ip mptcp endpoint add 10.1.0.2 signal
ip mptcp limits set add_addr_accepted 1 subflows 1
```

**node1 (client)** — set limits and pin the second local path:

```bash
ip mptcp limits set add_addr_accepted 1 subflows 1
ip mptcp endpoint add 10.1.0.1 dev eth3 subflow
```

| Endpoint type | Configured on    | Meaning                                                      |
| ------------- | ---------------- | ----------------------------------------------------------- |
| `signal`      | server (`node2`) | "I'm also reachable on this extra IP — connect to it."      |
| `subflow`     | client (`node1`) | "Use this local interface to open the extra path."          |

Test (ordinary apps aren't MPTCP-aware, so wrap them with `mptcpize`):

```bash
# node2
ip mptcp monitor          # watch CREATED / ESTABLISHED / ANNOUNCED / subflow events
mptcpize run iperf3 -s

# node1 — bandwidth doubles on a single session
mptcpize run iperf3 -c 10.0.0.2
bmon -p eth2,eth3
```

---

## 4. Site-to-site VPN with sing-box

`mptcpize` per app doesn't scale. `sing-box` transparently terminates connections on a TUN interface and re-opens them over an MPTCP Shadowsocks tunnel — no app changes. The tunnel uses **Shadowsocks 2022 (BLAKE3)** with `tcp_multi_path` enabled.

Install sing-box (see the [docs](https://sing-box.sagernet.org/installation/package-manager/)), then make sure the MPTCP endpoints/limits from step 3 are in place on both nodes (that's what lets the tunnel aggregate):

```bash
# node2 (server side of the path)
ip mptcp endpoint add 10.1.0.2 signal
ip mptcp limits set add_addr_accepted 1 subflows 1

# node1 (client side of the path)
ip mptcp endpoint add 10.1.0.1 dev eth3 subflow
ip mptcp limits set add_addr_accepted 1 subflows 1
```

Download the configs (already in this repo):

```bash
# node1
wget https://raw.githubusercontent.com/filip-lebiecki/mptcp/main/config_node1.json -O /etc/sing-box/config.json
# node2
wget https://raw.githubusercontent.com/filip-lebiecki/mptcp/main/config_node2.json -O /etc/sing-box/config.json
```

Config highlights:
- `inbounds[0]` — Shadowsocks listener (`tcp_multi_path: true`). Its password must match the **other** node's outbound password.
- `inbounds[1]` — `tun0` with `auto_route: false` (we add routes ourselves) and the gVisor stack.
- `outbounds[0]` — Shadowsocks to the peer (`tcp_multi_path: true`, `udp_over_tcp: true` to carry UDP inside the TCP stream).
- `route` — traffic to the remote LAN → `ss-out`; private/local traffic → `direct`. `auto_detect_interface: false` (prevents the direct outbound from binding to the wrong egress, which can drop UDP destined to the node's own LAN IP).

Two tunnels (one per direction) give full site-to-site reachability.

Start it and add the LAN route via the tunnel:

```bash
# node1
systemctl restart sing-box
ip route add 172.16.1.0/24 dev tun0

# node2
systemctl restart sing-box
ip route add 172.16.0.0/24 dev tun0
```

Test — full aggregated bandwidth on a **single, plain** TCP session (no `mptcpize` needed; sing-box does the MPTCP):

```bash
# node2
iperf3 -s
# node1
iperf3 -c 172.16.1.1
bmon -p eth2,eth3
ss -Mi            # inspect the subflows (or: sstui)
```

UDP works too (encapsulated via `udp_over_tcp`). Keep datagrams under the path MTU so they don't fragment:

```bash
iperf3 -c 172.16.1.1 -u -b 100M -l 1400
```

Failover — drop a link mid-transfer and traffic keeps flowing:

```bash
ip link set eth2 down
```

---

## 5. Both nodes behind NAT (public IPs)

The most realistic case: two dual-homed nodes, each behind 1:1 NAT, with **no public IP on the host itself**.

```
 node1                          upstream NAT                          node2
 eth0 192.168.80.30  ⇄  203.0.113.1   ──internet──   198.51.100.1  ⇄  192.168.80.13  eth0
 eth1 192.168.12.100 ⇄  203.0.113.2                  198.51.100.2  ⇄  192.168.12.101 eth1
```

> The links are now `eth0`/`eth1`. Start from a clean slate (reboot) so no `10.x` endpoints/routes linger.
> `192.168.80.1` / `192.168.12.200` below are the per-link upstream gateways — replace with yours.

**node1** — pin each peer IP to its own uplink, advertise the 2nd public IP, raise limits:

```bash
ip route add 198.51.100.1/32 via 192.168.80.1   dev eth0
ip route add 198.51.100.2/32 via 192.168.12.200 dev eth1
ip mptcp endpoint add 203.0.113.2 signal
ip mptcp limits set add_addr_accepted 1 subflows 2
```

**node2** — same idea, mirrored:

```bash
ip route add 203.0.113.1/32 via 192.168.80.1   dev eth0
ip route add 203.0.113.2/32 via 192.168.12.200 dev eth1
ip mptcp endpoint add 198.51.100.2 signal
ip mptcp limits set add_addr_accepted 1 subflows 2
```

Notes:
- **No source policy-routing needed** here — the two subflows go to two *different* destination IPs, each already pinned to its own interface by the routes above. (Source-based routing is only required when both subflows share one destination IP.)
- **You can advertise a public IP the host doesn't own.** `203.0.113.2` lives on the NAT, not on any interface — `signal` is just a hint telling the peer where to connect, and the NAT delivers the subflow back. No `ip_nonlocal_bind` or dummy interface required.
- **`subflows 2`, not `1`** — once a node both advertises its own address *and* opens a subflow toward the peer's, a limit of `1` silently drops the second path.

Point each `sing-box` outbound at the peer's **public** IP (edit `outbounds[0].server`):

```bash
# node1: "server": "198.51.100.1"     # node2's public IP
# node2: "server": "203.0.113.1"      # node1's public IP
```

Then restart and re-add the tunnel routes:

```bash
# node1
systemctl restart sing-box
ip route add 172.16.1.0/24 dev tun0
# node2
systemctl restart sing-box
ip route add 172.16.0.0/24 dev tun0
```

Test:

```bash
# node2
iperf3 -s
# node1
iperf3 -c 172.16.1.1
bmon -p eth0,eth1
```

Full aggregation across both paths — even though both servers are behind NAT.

---

## Notes & limitations

- **This is a Layer-4 proxy, not a Layer-3 VPN.** It carries **TCP and UDP only**. ICMP (ping/traceroute), GRE, ESP/IPsec, OSPF and raw IP do **not** traverse it.
- **UDP** is tunneled via `udp_over_tcp`. With the gVisor TUN's large MTU, use a sane datagram size (`iperf3 ... -u -l 1400`) so traffic isn't fragmented and lost.
- **Persistence:** all `ip route` / `ip mptcp` / `sysctl` commands here are runtime-only and lost on reboot. Make them persistent (e.g. systemd unit, networkd/netplan) for production. The sing-box config on disk is persistent; just re-add the `tun0` route after each `sing-box` restart (it recreates `tun0`).
- **Middleboxes:** MPTCP advertises its capabilities via TCP options. If a firewall strips them, the connection gracefully falls back to plain single-path TCP. When you control both ends (as here), it works reliably.
- **One-way tunnel:** if you only need connections initiated from one side, a single tunnel is enough. The reverse direction simply won't be reachable.

---

## Files

| File | Purpose |
| --- | --- |
| [`config_node1.json`](config_node1.json) | sing-box config for **node1** (base connects to `10.0.0.2`; edit `server` to node2's public IP for the NAT chapter) |
| [`config_node2.json`](config_node2.json) | sing-box config for **node2** (base connects to `10.0.0.1`; edit `server` to node1's public IP for the NAT chapter) |
