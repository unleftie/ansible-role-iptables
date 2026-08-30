# Compatibility with Docker

## `FORWARD` policy `DROP` is safe with Docker

[tasks/policies.yml](../tasks/policies.yml) sets the `FORWARD` chain policy to `DROP`.
This matches Docker's own documented pattern: dockerd inserts its own jump rules
(`DOCKER-USER`, `DOCKER-ISOLATION-STAGE-1/2`, `DOCKER`) at the **top** of `FORWARD`
whenever it starts. Chain policy is only consulted when no rule matches, so Docker's
explicit ACCEPT/jump rules take effect regardless of this role's policy or run order:

- Role runs first, Docker installed later → dockerd's insert lands above this role's rules.
- Docker already running, role re-converges → this role's `FORWARD` inserts (only created
  when `iptables_ipv4_nat`/`iptables_ipv6_nat` is `true`, see [nat.md](nat.md)) land above
  Docker's, but Docker's rule is still reachable for anything the narrower rule above it
  doesn't match.

Docker also manages its own `MASQUERADE` rule in the `nat` table for its bridge subnet
independently — container internet egress does not depend on `iptables_ipv4_nat`/
`iptables_ipv6_nat` at all.

## Published container ports bypass the `INPUT` chain

This role's `INPUT` chain only accepts loopback, established/related, ssh, icmp, and
dhcp — everything else hits the `DROP` policy. **This does not protect Docker's
published ports.** Docker DNATs that traffic in `PREROUTING` before the routing decision,
so it flows through `FORWARD` into the container's network namespace, never through
`INPUT`. Anything published with `ports:`/`-p` is reachable from wherever Docker's own
rules allow — by default, from anywhere — regardless of this role's `INPUT` rules.
Restricting access to published ports requires rules in the `DOCKER-USER` chain, which
this role does not manage.

## `iptables_flush: true` can break Docker on an already-dockerized host

[tasks/flush.yml](../tasks/flush.yml) (only runs when `iptables_flush: true`) flushes
`FORWARD`, which removes Docker's `DOCKER-USER`/`DOCKER-ISOLATION-*` jump rules, and then
deletes now-unreferenced custom chains via `iptables -X`. On a host that already has
Docker running, this deletes Docker's entire chain structure — containers keep running,
but published ports, inter-container communication, and NAT'd egress break until
`dockerd` is restarted (it only recreates these chains at daemon start or certain network
lifecycle events, not automatically when they go missing).

`iptables_flush` defaults to `false` and is not set anywhere in this repo, so normal
converges are unaffected. Avoid setting it to `true` on a host that already runs Docker;
if you must, restart the `docker` service afterward.
