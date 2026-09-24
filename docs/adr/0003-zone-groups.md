# Zone Groups for Multi-Zone Rules and Forwardings

Upstream OpenWrt firewall4 only supports individual zone references in `src` and `dest` options of rules, forwardings, redirects, and NAT entries, forcing administrators to write repetitive rules for every zone combination in multi-VLAN, IoT, and VPN setups. We decided to introduce `config zone_group` as a first-class named membership container in `/etc/config/firewall`, supporting Zone Groups and multi-valued `src`/`dest` lists across `config rule`, `config forwarding`, `config redirect` (source and reflection_zone), and `config nat`. These references are expanded at parse-time into the existing per-zone nftables chains (`forward_<zone>`, `input_<zone>`), ensuring full backward compatibility, exact document rule ordering, and inter-member forwarding semantics.

## Considered Options

- **Composite Jump Chains (`accept_to_<group>`)**: Create intermediate jump chains for groups. Rejected because it only optimizes the egress jump, cannot simplify ingress without disrupting zone-specific rule order, and adds unnecessary nftables chain overhead.
- **Native Interface Sets in Base Chains**: Define nftables sets for group devices and place rules in base `chain forward`. Rejected because it bypasses zone-specific chains, breaking per-zone MSS clamping, connection tracking helpers, invalid packet dropping, and zone default policies.
- **Full-Mesh Intra-Zone Expansion ($A \to A$)**: In intra-group forwarding (`src: 'group'`, `dest: 'group'`), automatically forward traffic within the same zone. Rejected because it would override a zone's own `option forward 'DROP'` policy for internal traffic; intra-zone routing remains solely governed by each member zone's own forward policy.
- **Group Inversion (`!group`)**: Complement expansion to all zones not in the group. Rejected to prevent accidentally routing or exposing sensitive zones (such as WAN or guest networks) through open-ended negative matching.
- **Parse-Time Macro Expansion (Selected)**: Flatten group references into individual zone instances during UCI parsing in `fw4.uc`. This preserves exact rule evaluation order, works seamlessly with existing verdict chains, and requires zero modifications to underlying nftables templates (`ruleset.uc`, `rule.uc`, `zone-verdict.uc`).

## Consequences

- `fw4.uc` parses `config zone_group` sections immediately after logical zones and before rules, forwardings, redirects, and NAT configurations.
- `state.zone_groups` tracks validated groups; group names and zone names share a unified namespace with collision detection.
- Options `src` and `dest` in `config rule` and `config forwarding` accept lists (`PARSE_LIST`) containing concrete zone names, zone groups, or a mix thereof.
- In `config redirect`, `src` and `reflection_zone` support zone groups (for multi-WAN ingress and multi-LAN reflection), but `dest` is restricted to concrete zones to ensure deterministic DNAT host routing.
- Forwardings and dual-zone SNAT expand to inter-zone pairs ($A \ne B$), omitting intra-zone self-forwarding and self-NAT ($A \to A$).
- Generated nftables rules include contextual comments: `!fw4: <name> [<concrete_src> -> <concrete_dest>] (<group_context>)`.
- Syslog log prefixes (`rule.log`) retain the clean base rule name (`<name>: `) to avoid kernel log prefix truncation.
