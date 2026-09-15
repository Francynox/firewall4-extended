# Specific Per-DNAT Filter Rules

Upstream OpenWrt firewall4 permits all destination NAT traffic via blanket `ct status dnat accept` rules emitted in both `input_<zone>` and `forward_<zone>` whenever a zone has any DNAT redirect. We decided to replace these blanket rules with granular, per-DNAT filter rules matching the post-translation destination address, port, protocol, source constraints, and egress zone (`jump accept_to_<dest>`), separating Port Redirections (`input`) from Port Forwards (`forward`) and supporting an opt-out via `option filter_rule '0'`.

## Considered Options

- **Blanket `ct status dnat accept` (Upstream default)**: Minimal ruleset size and automatically accepts non-UCI DNAT, but prevents decoupling DNAT from filtering (`option filter_rule '0'`), bypasses `accept_to_<dest>` egress zone enforcement, and opens both `input_<zone>` and `forward_<zone>` whenever either is used.
- **Strict Decoupling without Auto-Generation**: DNAT only configures translation in `dstnat`; users must manually create matching `config rule` entries in `forward`/`input`. Rejected as it breaks the zero-conf convenience expected by OpenWrt users.
- **Paired Specific Filter Rules with Opt-Out (Selected)**: Automatically generate specific `input_<src>` or `forward_<src>` filter rules (placed after user `config rule` entries so user `DROP`/`REJECT` overrides still win), with `option filter_rule '0'` to opt out per redirect.

## Consequences

- `ruleset.uc` no longer emits blanket `ct status dnat accept` rules in `input_<zone>` and `forward_<zone>`.
- Each `config redirect` (with `filter_rule` enabled, the default) generates its corresponding post-DNAT filter rule(s) in `input_<src>` (for Port Redirections targeting omitted `dest_ip`, loopback, or a local netifd interface IP) or `forward_<src>` (jumping to `accept_to_<dest>`, or `accept` when no destination zone exists).
- Cross-zone NAT reflection emits specific filter rules in each reflection zone (`forward_<rzone>` or `input_<rzone>`). Same-zone redirects (`rzone == src`) rely on the main redirect's filter rule so source exclusions are never bypassed.
- Source constraints (`src_ip`, `src_port`, `src_mac`, schedules, and source-only `ipset` matches) are preserved on the generated filter rule. Destination-matching `ipset` fields (`daddr`/`dport`) are intentionally omitted post-DNAT so rewritten packets are not falsely dropped.
- Redirect logging (`log` / `log_limit`) stays exclusively on `dstnat` to avoid double-logging every packet and consuming 2x `log_limit` budget.
- Identical post-DNAT filter rules are deduplicated across all packet-matching criteria (including `ipset` inversion/fields and `utc_time`) to avoid dead shadow rules in nftables.
- **Trade-offs / Limitations**:
  - Non-UCI DNAT (such as custom `/etc/nftables.d/` `dstnat_*` rules, container runtimes, or UPnP/NAT-PMP daemons that do not inject filter rules into `table inet fw4`) is no longer accepted automatically and requires an explicit filter rule or chain include.
  - Local router IPs not managed by netifd (e.g. Keepalived VIPs added via `ip addr`) cannot be auto-classified as `input`; use `option filter_rule '0'` paired with an explicit `input` `config rule` for those redirects.
