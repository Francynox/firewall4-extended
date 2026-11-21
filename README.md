# OpenWrt firewall4-extended

An enhanced replacement for OpenWrt's [`firewall4`](https://github.com/openwrt/firewall4) (nftables firewall). Adds first-class **Zone Groups**, multi-valued zone lists, granular per-DNAT filter rules, and decoupled NAT reflection.

Backward-compatible with standard UCI `/etc/config/firewall` configurations (see [Breaking Changes & Caveats](#breaking-changes--caveats) regarding non-UCI DNAT).

---

## Key Enhancements

### 1. Zone Groups & Multi-Zone Lists
Group multiple zones under `config zone_group` or specify zone lists directly in `src` and `dest` options across rules, forwardings, redirects, and NAT.

- Eliminates repetitive rules across multi-VLAN, IoT, and multi-WAN setups.
- Expands at parse time into existing per-zone nftables chains (`forward_<zone>`, `input_<zone>`, `output_<zone>`, `dstnat_<zone>`, `srcnat_<zone>`).
- Forwardings and dual-zone NAT expand to distinct inter-zone pairs (`A ≠ B`), omitting intra-zone self-forwarding and self-NAT (`A → A`).
- Dual-zone `config nat` (`src` + `dest`) matches ingress via `iifname` inside `srcnat_<dest>` (or base `srcnat` when `dest '*'`).
- `fw4 zone <group_name> [<device>]` resolves devices across member zones.

See [ADR 0003: Zone Groups](docs/adr/0003-zone-groups.md).

### 2. Granular DNAT Filter Rules
Replaces upstream's blanket `ct status dnat accept` rules in `input_<zone>` and `forward_<zone>` with specific post-DNAT filter rules per configured redirect:

- **Port Redirections** (router-local targets) accept specifically in `input_<src>`.
- **Port Forwards** (internal hosts) jump specifically to `accept_to_<dest>` in `forward_<src>`, enforcing egress zone matching and counters.
- Preserves source constraints (`src_ip`, `src_port`, `src_mac`, schedules, and source-only `ipset` matches) while omitting destination-matching `ipset` fields post-DNAT and keeping logging exclusively on `dstnat`.
- Add `option filter_rule '0'` per redirect to decouple DNAT from filtering and manage acceptance with explicit `config rule` entries.

See [ADR 0002: Specific DNAT Filter Rules](docs/adr/0002-specific-dnat-filter-rules.md).

### 3. Decoupled NAT Reflection
Decouples NAT reflection from ingress zone masquerading (`masq`/`masq6`) via tri-state `option reflection` (`1`, `0`, or unset/auto) on both `config zone` and `config redirect`:

- **Redirect `option reflection`**: `'0'` always disables; explicit `'1'` or a set `reflection_zone` always enables regardless of `masq`/`masq6`; unset defers to the source zone.
- **Zone `option reflection`**: optional `'1'`/`'0'` overrides the zone's default reflection policy for both IPv4 and IPv6 (e.g. routed WANs, VIPs, VPNs, or IPv6 DNAT); unset falls back to upstream `masq` / `masq6`.
- **Same-zone hairpin SNAT (`src == dest`)**: emits only `ct status dnat` SNAT in `srcnat_<zone>` without duplicating `dstnat`, allowing single-section `lan → lan` redirects (such as transparent DNS hijacks with `src_ip '!...'` and `src_dip '!...'`) without manual `config nat` rules.
- **Negated `src_dip` guard**: skips unsafe cross-zone reflection when `src_dip` is negated.

See [ADR 0001: Decouple NAT Reflection](docs/adr/0001-decouple-nat-reflection-from-zone-masquerade.md).

---

## Breaking Changes & Caveats

- **Non-UCI DNAT**: Removing blanket `ct status dnat accept` means non-UCI DNAT rules (custom `/etc/nftables.d/*.nft` `dstnat_*` rules, container runtimes, or UPnP/NAT-PMP daemons that do not inject filter rules into `table inet fw4`) are no longer accepted automatically and require an explicit `config rule` or chain include.
- **Non-netifd local IPs**: Local router addresses not managed by netifd (e.g. Keepalived VIPs added via `ip addr`) cannot be auto-classified as `input`; use `option filter_rule '0'` paired with an explicit `input` `config rule`.

---

## Configuration Examples

### Zone Groups & Forwarding

```uci
config zone_group
	option name 'trusted'
	list zone 'lan'
	list zone 'guest'

config zone_group
	option name 'wans'
	list zone 'wan'
	list zone 'wan6'

# Forward trusted zones to all WANs (lan->wan, lan->wan6, guest->wan, guest->wan6)
config forwarding
	option src 'trusted'
	option dest 'wans'

# Inter-VLAN mesh: forwards lan<->guest, skips self-forwarding (lan->lan, guest->guest)
config forwarding
	option src 'trusted'
	option dest 'trusted'
```

### Multi-WAN Port Forward with Multi-LAN Reflection

```uci
config redirect
	option name 'HTTP Server'
	option src 'wans'
	option dest 'lan'
	option proto 'tcp'
	option src_dport '80'
	option dest_ip '192.168.1.100'
	option dest_port '8080'
	option target 'dnat'
	list reflection_zone 'trusted'
```

### Routed WAN Reflection & Same-Zone LAN DNS Hijack

```uci
# Enable default reflection on an unmasqueraded routed WAN zone
config zone
	option name 'wan'
	list network 'wan'
	list network 'wan6'
	option masq '0'
	option reflection '1'

# Single-section LAN DNS hijack to internal resolver (hairpin SNAT via ct status dnat)
config redirect
	option name 'Force LAN DNS to Pi-hole'
	option src 'lan'
	option dest 'lan'
	option proto 'tcp udp'
	option src_ip '!192.168.1.53'
	option src_dip '!192.168.1.1'
	option src_dport '53'
	option dest_ip '192.168.1.53'
	option dest_port '53'
	option target 'dnat'
	option reflection '1'
```

---

## Installation

### Method 1: Drop-in replacement on a running router
Copy the modified ucode files and templates to your OpenWrt router:

```bash
scp root/usr/share/ucode/fw4.uc root@192.168.1.1:/usr/share/ucode/fw4.uc
scp root/usr/share/firewall4/main.uc root@192.168.1.1:/usr/share/firewall4/main.uc
scp root/usr/share/firewall4/templates/{redirect,rule,ruleset}.uc root@192.168.1.1:/usr/share/firewall4/templates/
ssh root@192.168.1.1 "fw4 reload"
```

### Method 2: OpenWrt Buildroot
Point `package/network/config/firewall4/Makefile` to this repository:

```makefile
PKG_SOURCE_PROTO:=git
PKG_SOURCE_URL:=https://github.com/francynox/firewall4-extended.git
PKG_SOURCE_VERSION:=<commit-hash>
```

---

## Tests

Run the full test suite locally (requires `ucode`):

```bash
./run_tests.sh
```

---

## License

ISC
