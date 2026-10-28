# Architecture

## Firewalls and zones

| Firewall | Interface | Zone | Address | VMware network |
|---|---|---|---|---|
| HQ | em0 | WAN | 192.168.23.129/24 (DHCP) | vmnet8 (NAT) |
| HQ | em1 | LAN / management | 10.10.10.254/24 | vmnet2 |
| HQ | em2 | SERVERS | 10.10.20.1/24 | vmnet3 |
| HQ | em3 | WORKSTATIONS | 10.10.30.1/24 | vmnet4 |
| HQ | em4 | REDZONE | 10.10.40.1/24 | vmnet5 |
| Branch | em0 | WAN | 192.168.23.128/24 (DHCP) | vmnet8 (NAT) |
| Branch | em1 | LAN | 10.10.50.254/24 | vmnet6 |
| Tunnel | wg0 | WireGuard | HQ 10.99.0.1, Branch 10.99.0.2 (/30) | over the WAN addresses |

Screenshots: [HQ interfaces](../evidence/00a-hq-interfaces.png), [Branch interfaces](../evidence/00b-branch-interfaces.png).

## Design choices

- **Address collisions with the host.** VMware gives its own host adapter the `.1` address on each host-only network. HQ's LAN uses `.254` to avoid that. The other zones use `.1`, and on vmnet5 the host adapter did collide with the firewall, see [troubleshooting.md](troubleshooting.md).
- **A second full OPNsense VM was used as the Branch** instead of a single remote peer, so the tunnel has routing and firewall policy at both ends.
- **WireGuard was chosen over IPsec** for simpler configuration and debugging. OPNsense 26.7 ships WireGuard in the core, so no plugin is needed.
- **Management access** to each firewall GUI is through the host's own adapter on the relevant network: `10.10.10.1` on vmnet2 for HQ and `10.10.50.1` on vmnet6 for the Branch.
- **DNS and DHCP** on HQ are served by Dnsmasq for the four internal zones only. WAN is excluded on purpose. The Branch LAN has no DHCP server.
