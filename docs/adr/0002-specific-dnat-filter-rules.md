# Specific Per-DNAT Filter Rules

Upstream OpenWrt firewall4 permits all destination NAT traffic via blanket `ct status dnat accept` rules in zone input and forward chains. I decided to replace these blanket rules with granular, per-DNAT filter rules matching the post-translation destination address, port, protocol, and egress zone (`jump accept_to_<dest>`), while separating Port Redirections (`input`) from Port Forwards (`forward`). This eliminates the security blast radius of unintentionally accepting non-UCI DNAT flows, enables per-rule filter accounting and logging, and preserves user DROP override capability while supporting an opt-out via `option filter_rule '0'`.

## Considered Options

- **Blanket `ct status dnat accept` (Upstream default)**: Single rule per zone; minimal ruleset complexity, but allows any DNAT from any source/daemon to bypass forward/input filtering and prevents per-redirect filter accounting.
- **Strict Decoupling without Auto-Generation**: DNAT only configures translation in `dstnat`; users must manually create matching `config rule` entries in forward/input. Rejected as it breaks the zero-conf convenience expected by OpenWrt users.
- **Paired Specific Filter Rules (Selected)**: Automatically generate matching specific forward/input filter rules for each `config redirect`, with support for NAT reflection and an explicit suppression toggle (`option filter_rule '0'`).

## Consequences

- `ruleset.uc` no longer emits blanket `ct status dnat accept` rules at lines 228 and 249.
- Each `config redirect` generates both the prerouting DNAT rule and one or more corresponding filter acceptance rules in the appropriate `input` or `forward` chains.
- NAT reflection automatically emits specific filter acceptance rules for all designated reflection zones (in `forward` for internal hosts, and in `input` for services hosted directly on the router).
- Existing tests expecting `ct status dnat accept` must be updated to reflect the new granular rules.
- When a redirect specifies source restrictions (e.g. source IP, MAC, ipset, or schedule), the generated filter rule preserves these match criteria so that access restrictions are enforced at the filter layer as well as the NAT layer.

