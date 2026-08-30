# NAT rules

`iptables_ipv4_nat` / `iptables_ipv6_nat` (default: `false`) turn the instance into a
router/NAT gateway for traffic it **forwards** on behalf of other hosts. When enabled,
[tasks/rules.yml](../tasks/rules.yml) adds:

1. `FORWARD` accept for `ESTABLISHED,RELATED` (inserted at position 1)
2. `FORWARD` accept for `NEW` connections from `iptables_ipv4_nat_source_pool` /
   `iptables_ipv6_nat_source_pool`
3. `POSTROUTING -j MASQUERADE` in the `nat` table, out the default-route interface

This only affects packets the host **forwards**, not traffic it originates itself —
locally-originated packets go `OUTPUT → POSTROUTING` with the host's own address already
as source, so MASQUERADE is a no-op for them and they never touch `FORWARD`.

## Do you need it?

**No, on a plain AWS/GCP/Hetzner instance.** All three providers already do 1:1 NAT
between the private IP and the public/elastic/floating IP at the platform level, outside
the guest OS. Enabling MASQUERADE there adds nothing.

**Yes, when the instance routes traffic for something else**, e.g.:

- a VPN server (WireGuard/OpenVPN) — client tunnel addresses don't exist in the
  provider's network, so return traffic needs masquerading to the server's IP
- a self-hosted NAT gateway for private-subnet instances
- (not Docker — dockerd manages its own MASQUERADE for the bridge subnet independently
  of this toggle; see [docker-compatibility.md](docker-compatibility.md))

## Prerequisites the role does not manage

- **`net.ipv4.ip_forward=1`** must be set via sysctl — not currently done anywhere in
  this role. Without it the kernel drops forwarded packets before iptables ever runs.

## Default value caveats

- `iptables_ipv4_nat_source_pool` defaults to `iptables_internal_ip_pool`
  (`10.0.0.0/16`). Override it to match the actual client/tunnel subnet before enabling
  NAT — the internal pool is not the same thing.
- `iptables_ipv6_nat_source_pool` defaults to `::/0` (masquerade everything). NAT66 is
  generally an anti-pattern — cloud providers hand out routed IPv6 blocks per instance,
  so prefer routing + firewalling IPv6 clients over NAT. Leave `iptables_ipv6_nat: false`
  unless you have a specific reason not to.
