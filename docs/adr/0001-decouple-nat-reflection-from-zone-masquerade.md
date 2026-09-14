# Decouple NAT Reflection from Ingress Zone Masquerade

Upstream OpenWrt firewall4 conditioned NAT reflection rule generation on whether the source (ingress) zone has outbound masquerading enabled (`src.zone.masq` for IPv4, `src.zone.masq6` for IPv6). Even when a user explicitly configured `option reflection '1'` or `reflection_zone`, reflection was silently dropped on unmasqueraded source zones such as routed IPv4 WANs, Keepalived VIP topologies, VPN ingress zones, same-zone LAN redirects, and IPv6 DNAT redirects.

We decided to make `option reflection` tri-state (`1`, `0`, or unset/auto) on both `config redirect` and `config zone`, decoupling explicit reflection from `masq`/`masq6` while keeping unset/auto behavior 100% backward-compatible with upstream OpenWrt configurations, supporting clean SNAT-only hairpin reflection for same-zone redirects (`rzone == src.zone`), and blocking unsafe cross-zone reflection on negated `src_dip` transparent redirects.

## Considered Options

- **Coupled to Source Zone Masquerade (Upstream default)**: Emits reflection rules only if the ingress zone has `masq: 1` (or `masq6: 1`). Rejected because it silently ignores explicit reflection requests on routed public subnets, VIP topologies, VPN zones, same-zone DNAT redirects, and IPv6 DNAT redirects.
- **Unconditionally Always-On Reflection (Drop `masq` check with default `reflection '1'`)**: Rejected because internal DNAT redirects (such as `lan -> lan` or `guest -> lan` transparent DNS/proxy interception) omit `option reflection` and would silently emit reflection DNAT rules, causing forwarding loops on self-redirects and hijacking the reflection zone when `src_dip` is negated.
- **Tri-State Zone & Redirect Reflection with Same-Zone Hairpin SNAT and Negated `src_dip` Guard (Selected)**:
  - `config redirect`: `option reflection '0'` always disables reflection; explicit `option reflection '1'` or a configured `reflection_zone` (with `reflection != '0'`) always enables reflection regardless of `masq`/`masq6`; omitted `reflection` defers to the source zone.
  - `config zone`: optional `option reflection '1'|'0'` overrides the zone's default reflection policy for both IPv4 and IPv6; omitted `reflection` falls back to `masq` (IPv4) / `masq6` (IPv6).
  - **Same-zone redirects (`rzone == src.zone`)**: Skip redundant reflection DNAT (and reflection filter rules), since the main redirect rule already runs in `dstnat_<src>` with full `src_ip`/`src_dip`/`src_mac`/`ipset` constraints, and emit only the hairpin SNAT rule in `srcnat_<rzone>` scoped with `ct status dnat`. This enables single-section `lan -> lan` redirects and transparent DNS hijacks without manual `config nat` rules.
  - **Negated `src_dip` guard**: Skip cross-zone reflection (and same-zone reflection with `reflection_src 'external'`) when `src_dip` is negated, warning if reflection was explicitly requested.

## Consequences

- Existing upstream `/etc/config/firewall` configurations behave identically with zero surprise reflection on internal redirects.
- Routed IPv4 WANs, VPN ingress zones, and IPv6 DNAT redirects can enable reflection either per-redirect (`option reflection '1'` or `reflection_zone`) or once at the zone level (`option reflection '1'` on `config zone`).
- Masqueraded zones can opt out of default reflection via `option reflection '0'` on `config zone` while still allowing per-redirect overrides.
- Same-zone redirects (`src == dest`, e.g. `lan -> lan` DNS hijack to an internal resolver) with `option reflection '1'` emit `ct status dnat` hairpin SNAT in `srcnat_<zone>` without shadowing the main DNAT rule.
- Upstream's eager `redir.dest.zone.dflags.{accept,dnat,snat}` assignment is removed; `dflags.dnat` and `dflags.snat` are set lazily on the actual reflection zone(s) in `add_rule()`, eliminating unused `accept_to_<dest>` and empty NAT chains.
- Covered and verified by `tests/03_rules/16_redirect_reflection`.
