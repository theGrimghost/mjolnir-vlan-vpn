# WireGuard site-to-site tunnel

## Address plan

| Item | HQ | Branch |
|---|---|---|
| WAN address | 192.168.23.129 | 192.168.23.128 |
| Tunnel address | 10.99.0.1/30 | 10.99.0.2/30 |
| Listen port | 51820/UDP | 51820/UDP |
| LAN behind it | 10.10.10.0/24 plus the SERVERS, WORKSTATIONS and REDZONE zones | 10.10.50.0/24 |

OPNsense 26.7 has WireGuard in the core. The firmware plugin search shows only `os-tailscale`, which is a different product, so nothing is installed.

## 1. Instances (VPN > WireGuard > Instances)

Each side gets one instance with a generated key pair, the listen port and its tunnel address. Public keys are exchanged between the sides. Private keys never leave the firewall, and the screenshot in this repo has the private key redacted.

| Setting | HQ (`tunnel-to-branch`) | Branch (`tunnel-to-hq`) |
|---|---|---|
| Listen port | 51820 | 51820 |
| Tunnel address | 10.99.0.1/30 | 10.99.0.2/30 |
| Peers | `peer-branch` | `peer-hq` |

Screenshot: [HQ instance](../evidence/01-hq-wireguard-instance.png).

## 2. Peers (VPN > WireGuard > Peers)

The peer's **Allowed IPs** control both what is accepted from that peer and which routes OPNsense installs, so they must list every remote subnet that needs to cross the tunnel.

| Peer | Public key | Allowed IPs | Endpoint |
|---|---|---|---|
| HQ `peer-branch` | Branch public key | 10.99.0.2/32, 10.10.50.0/24 | 192.168.23.128:51820 |
| Branch `peer-hq` | HQ public key | 10.99.0.1/32, 10.10.10.0/24, 10.10.20.0/24, 10.10.30.0/24, 10.10.40.0/24 | 192.168.23.129:51820 |

Screenshots: [HQ peer](../evidence/02-hq-wireguard-peer.png), [Branch peer](../evidence/03-branch-wireguard-peer-allowed-ips.png).

Resulting routes: HQ sends `10.10.50.0/24` via `wg0` ([screenshot](../evidence/06-hq-routes.png)), and the Branch sends `10.10.10.0/24` to `10.10.40.0/24` via `wg0` ([screenshot](../evidence/07-branch-routes.png)).

## 3. Firewall rules

**WAN, both firewalls.** Pass, IPv4, UDP, destination port 51820. Without it the handshake never reaches the WireGuard daemon. In the rule editor, the Destination field is an alias search box, so the port goes in **Destination Port** (Single port or range). Screenshot: [WAN rules](../evidence/04-wan-udp-51820-rules.png).

**WAN, both firewalls.** *Block private networks* and *Block bogon networks* are disabled, because the WAN here is a private VMware NAT network.

**WireGuard group, final state**

| Firewall | Protocol | Source | Destination | Purpose |
|---|---|---|---|---|
| HQ | any | 10.10.50.0/24 | 10.10.10.0/24 | Branch LAN to HQ LAN only |
| HQ | ICMP | 10.99.0.0/30 | 10.99.0.0/30 | Tunnel health checks |
| Branch | any | 10.10.10.0/24 | 10.10.50.0/24 | HQ LAN to Branch LAN |
| Branch | any | 10.10.40.0/24 | 10.10.50.0/24 | REDZONE to Branch LAN |
| Branch | ICMP | 10.99.0.0/30 | 10.99.0.0/30 | Tunnel health checks |

Screenshots: [HQ](../evidence/10-wg-rules-after-hq.png), [Branch](../evidence/11-wg-rules-after-branch.png).

The tunnel started with any-to-any pass rules. Those were kept alongside the scoped rules at first, which made the scoped rules ineffective because OPNsense uses the first matching rule. They were disabled and then deleted. Before: [HQ](../evidence/08-wg-rules-before-hq.png), [Branch](../evidence/09-wg-rules-before-branch.png).
