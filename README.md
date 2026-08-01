# ansible-role-firewall

nftables host firewall with a data-driven allow-list, **validated before it is
applied**, and deliberately Docker-safe.

No packet filter existed anywhere on this fleet — a 2026-07-31 audit found no
nftables, iptables or ufw ruleset at all, so every listener was reachable from
the whole LAN. `devsec.hardening` ships no firewall role; `os_hardening` does
not cover this.

This is the highest-stakes role in the collection: a bad ruleset locks out the
fleet *and* the host applying it. The design reflects that.

## Requirements

Debian family.

## The four safety properties

**1. The ruleset is parsed before it is installed.** The template runs
`nft -c -f` on the candidate file, so a syntax error fails the play instead of
being written and then applied to a host you can no longer reach.

**2. SSH cannot be forgotten.** It's declared separately from `firewall_rules`,
allowed by default, and the role *asserts* it is permitted whenever the input
policy is `drop`. Turning it off has to be written down, not arrived at.

**3. Output stays `accept`.** Egress filtering breaks tunnelled access
(a Twingate connector), package updates, ACME renewals and image pulls — and it
breaks them from the outside in. Change it only with a tested allow-list.

**4. `established,related` and loopback are always accepted**, so a correct
ruleset never severs the connection applying it.

ICMP is allowed by default. On IPv6 that is not politeness: dropping ICMPv6
breaks neighbour discovery and PMTU, which presents as intermittent hangs rather
than a clean refusal.

## Docker safety

The generated `/etc/nftables.conf` **never contains `flush ruleset`.**

Docker installs its own nat/filter chains through the iptables-nft backend. A
global flush deletes them: containers keep running, published ports stop
answering, and nothing logs an error. This role instead recreates only its own
table (`firewall_table_name`, default `ansible_managed`) and leaves everything
else alone. The molecule suite asserts the absence of `flush ruleset` so nobody
reintroduces it.

Container traffic is unaffected by `input policy drop` regardless: published
ports arrive via nat/PREROUTING and traverse FORWARD, not INPUT. Restricting
container exposure is Docker's job (or a `DOCKER-USER` chain), not this table's.

## Role variables

| Variable | Default | Description |
|---|---|---|
| `firewall_input_policy` | `drop` | |
| `firewall_output_policy` | `accept` | See safety property 3. |
| `firewall_allow_ssh` | `true` | |
| `firewall_ssh_ports` | `[22]` | |
| `firewall_ssh_sources` | `[]` | Empty = anywhere. Set CIDRs for LAN hosts. |
| `firewall_rules` | `[]` | `port`, `proto` (default tcp), `sources`, `comment`. |
| `firewall_allow_icmp` / `_loopback` / `_established` | `true` | Leave on. |
| `firewall_table_name` | `ansible_managed` | The only table this role touches. |

## Two shapes

**LAN host** — no SSH reachable from the internet, so restrict it to the
networks you administer from:

```yaml
firewall_ssh_sources: [192.168.2.0/24, 192.168.3.0/24]
```

**Public VPS** — SSH must stay reachable from anywhere, because that is the only
way the host is managed:

```yaml
firewall_ssh_sources: []          # anywhere
firewall_rules:
  - {port: 80,  comment: http}
  - {port: 443, comment: https}
```

## Rollout

Never converge this fleet-wide first. `--check`, then `--limit` a single
low-value host, confirm SSH from a *new* connection (an existing session
survives on `established,related` and will lie to you), then expand.

## Testing

```bash
molecule test
```

Debian 12/13 and Ubuntu 24.04.

## License

MIT
