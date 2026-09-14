# Decouple NAT Reflection from Ingress Zone Masquerade

Upstream OpenWrt firewall4 conditioned NAT reflection rule generation on whether the source (ingress) zone has outbound masquerading enabled (`redir.src.zone[i ? "masq6" : "masq"]`). I decided to decouple NAT reflection from the source zone's masquerade flags, relying instead on the presence of target rewrite addresses and explicit reflection configuration. This allows NAT reflection to function on routed IPv4 subnets, double-NAT / Keepalived VIP upstream topologies, and non-masqueraded WAN interfaces, while also enabling IPv6 NAT reflection where masquerading (`masq6`) is standardly disabled.

## Considered Options

- **Coupled to Source Zone Masquerade (Upstream default)**: Emits reflection rules only if the ingress zone has `masq: 1` (or `masq6: 1`). Rejected because it permanently disables IPv6 reflection in standard routed environments and breaks IPv4 reflection for routed public subnets, VIP topologies, and edge routers where WAN masquerade is disabled.
- **Decoupled Reflection Generation (Selected)**: Emit NAT reflection DNAT and SNAT rules whenever target rewrite addresses exist and reflection is enabled (`reflection != 0`), irrespective of the ingress zone's masquerade status.

## Consequences

- `fw4.uc` (line 2996) evaluates rewrite target address availability (`length(rip[i])`) without checking `redir.src.zone.masq` or `redir.src.zone.masq6`.
- Port forwards configured on non-masqueraded WAN zones or IPv6 zones now reliably generate reflection rules in the target zone and any configured `reflection_zone`.
- Internal-to-internal redirects with `target 'dnat'` generate reflection rules by default unless suppressed with `option reflection '0'`.
- Covered and verified by test suite `tests/03_rules/16_redirect_reflection`.

