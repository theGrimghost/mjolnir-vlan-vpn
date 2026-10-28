# Zone segmentation

The HQ firewall is deny-by-default. Each internal zone has explicit block rules for the zones it must not reach, followed by a catch-all allow for outbound traffic. OPNsense is stateful, so return traffic is allowed automatically and no mirrored rules are needed.

## Policy (from the lab design notes)

| Source zone | Explicitly blocked destinations | Then |
|---|---|---|
| REDZONE | LAN, SERVERS, WORKSTATIONS | Catch-all allow (`Allow REDZONE to Internet`) |
| WORKSTATIONS | LAN, REDZONE | Catch-all allow |
| SERVERS | LAN, REDZONE | Catch-all allow |

The REDZONE catch-all is a pass rule from `REDZONE network` to any destination with logging enabled.

This table comes from the design notes. The full rule list was not re-exported or re-tested for this repo, and the verification in this repo covers the WireGuard path only.

## Design observation: the catch-all is broader than "internet"

Because the catch-all allows any destination that no earlier rule blocked, it also allows the Branch subnet and the tunnel. That is why Kali (REDZONE) can ping the Branch LAN in [verification.md](verification.md).

Whether that is acceptable depends on the intent of REDZONE. If REDZONE should be isolated from the Branch as well, add a block rule for `10.10.50.0/24` above the catch-all. This is recorded as an open decision, not a fixed result.
