# FastACL — Community Edition

A high-performance inline packet-filtering plugin for [FD.io VPP](https://fd.io/),
built for DDoS mitigation at 100 Gbps line rate. FastACL implements the
[RFC 8955](https://www.rfc-editor.org/rfc/rfc8955) (IPv4) and
[RFC 8956](https://www.rfc-editor.org/rfc/rfc8956) (IPv6) FlowSpec match model as
a VPP feature-arc node, designed to act as a scrubbing data plane driven by an
external detection system (for example FastNetMon) over VPP's CLI and binary API.

Released under the **Apache License 2.0** — the same licence as VPP itself.

Plugin documentation in VPP's own format, covering the classifier architecture,
is at [`src/plugins/fastacl/fastacl.rst`](src/plugins/fastacl/fastacl.rst) — the
location VPP expects a plugin's docs, so it works unchanged if the plugin is
built inside a VPP tree.

## Features

- **All 12 RFC 8955/8956 match types**, IPv4 and IPv6 (dual-stack): destination
  and source prefix, IP protocol, destination/source/either port ranges, ICMP
  type and code, TCP flags (value + bitmask), packet length, DSCP, and fragment
  flags.
- **`drop` and `permit` actions**, with the action type carried in the classifier
  result so the fast path never dereferences the rule pool to discard a packet.
  `permit` matches and forwards unchanged, which is what makes a sample-only rule
  possible; an action is mandatory, because type 0 is `drop` and an omitted one
  would silently discard the traffic a tap rule was added to observe.
- **Per-rule traffic sampling** — an exact (not statistical) 1-in-N ratio per
  rule, with the copy taken *before* the action is applied, so a rule can show
  what it discarded. Delivery is over the kernel
  [psample](https://www.kernel.org/doc/html/latest/networking/index.html)
  netlink channel, readable by any ordinary Linux program.
- **TSS classifier** (Tuple Space Search): rules are grouped by mask shape into
  per-shape VPP bihash tables, so lookup cost scales with the number of *distinct
  mask shapes* rather than the number of rules. Keys narrow to a compact 8-byte
  (IPv4) or 16-byte (IPv6) form when the mask allows.
- **Rule ordering** following RFC 8955 §5.1 component-wise precedence, with
  single and batch rule installation.
- **Dual-stack scoping that follows the rule, not the family.** A rule applies
  to the family its addresses name, and to *both* when it names no address —
  so `proto 17 dst-port 53 action drop` filters IPv4 and IPv6, rather than
  silently exempting IPv6.
- **Per-rule and aggregate counters**, including packet/byte totals and
  wire-rate telemetry (pps, L3 bps, and L1 bps accounting for framing overhead).
- **Routed and bridged operation** from one datapath: the filter attaches to the
  `ip4-unicast` / `ip6-unicast` arcs and to `l2-input-ip4` / `l2-input-ip6`.
- **VPP binary API and CLI** for every operation.

## Building

The plugin builds out-of-tree against installed VPP development packages.

```sh
# Prerequisites: VPP 25.10 or 26.06 dev packages (vpp-dev, libvppinfra-dev)
#                libnl-3 headers (libnl-3-dev, libnl-genl-3-dev) for psample
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
make -C build -j"$(nproc)"
```

The result is `build/lib/vpp_plugins/fastacl_plugin.so`. Point VPP at it with a
`plugin_path`, or copy it into VPP's plugin directory:

```
plugins {
  path /path/to/build/lib/vpp_plugins
  plugin fastacl_plugin.so { enable }
}
```

## Quick start

Enable the filter on an interface, then install rules:

```
set interface fastacl TenGigabitEthernet0/0/0

# Drop UDP/53 to a prefix
fastacl rule add order 10 dst 10.0.0.0/24 proto 17 dst-port 53 action drop

# Tap: match and forward, sampling 1 packet in 1000 (pass/accept also work)
fastacl rule add order 15 dst 10.0.0.0/24 action permit sample 1000

# Drop TCP SYN to an IPv6 prefix, sampling 1 packet in 100 before the drop
fastacl rule add order 20 dst6 2001:db8::/32 proto 6 \
    tcp-flags value 0x02 mask 0x02 action drop sample 100

show fastacl rules
show fastacl aggregate-counters
show fastacl tuples

# Per-rule counters are exact and on by default; turn the counter write off
fastacl per-rule-stats disable
fastacl per-rule-stats            # no argument reports the current state
```

`show fastacl tuples` reports how many distinct mask shapes are active — the
quantity that actually drives per-packet classification cost — along with
`chains` / `chained` / `deepest`, which say how many rules share a key and how
long the resulting walk can get.

To read sampled packets, enable the psample channel and attach any psample
consumer:

```
fastacl psample enable
show fastacl sampling
```

### Tuning

One startup-config knob sizes the classifier's hash tables. It is allocated per
rule *shape*, so raise it for large rule sets — a million-rule table runs
`1048576`. Values below `1024` are clamped up.

```
fastacl {
  tss-bihash-buckets 1048576
}
```

(`tss-bihash-memory-mb` is still parsed so old configurations start, but it does
nothing: the classifier's hash tables set `BIHASH_USE_HEAP` and allocate from
the VPP main heap. Size the main heap instead.)

The settings outside the `fastacl` stanza matter as much as the one inside it.
A 32-worker ConnectX-7 box carrying a million rules at 100 GbE runs:

```
memory  { main-heap-size 12G  main-heap-page-size 2M }
statseg { size 12G }
buffers { buffers-per-numa 2097152 }
cpu     { main-core 0  corelist-workers 1-31 }
dpdk {
  dev 0000:81:00.0 {
    num-rx-queues 32  num-tx-queues 32
    num-rx-desc 4096  num-tx-desc 4096
    rss { ipv4 ipv6 l3-src-only }
  }
}
```

- `num-rx-desc` is the one that surprises people. Large RX buffers evict the
  rule table from the last-level cache and every per-rule counter write becomes
  a cache miss; 4096 measures an order of magnitude less ingress loss than 8192
  at a million rules. Nothing in the plugin sets it.
- `statseg size` has to cover the per-rule counters —
  `rules x (workers + 1) x 16` bytes.
- `main-heap-size` has to cover the classifier: every tuple's hash table comes
  off the main heap. 2 MB pages are worth setting.
- `rss { ipv4 ipv6 }` is not optional for dual-stack work. Without `ipv6` in the
  list the NIC hashes the whole family to one queue, and one worker.

## Performance notes

Classification cost is a function of the number of **distinct mask shapes**, not
the number of rules — a thousand rules that share one shape cost roughly the
same as one rule. A rule set spread across many shapes costs proportionally more,
because each shape is a separate hash probe. When sizing a deployment, count
shapes with `show fastacl tuples` rather than counting rules.

Sustaining 100 Gbps at small frame sizes requires the usual VPP data-plane
hygiene: multiple worker threads each pinned to a dedicated core, RSS spreading
traffic across as many receive queues as there are workers, isolated cores
(`isolcpus`, `nohz_full`, `rcu_nocbs`), and C-states limited.

## Relationship to the commercial edition

This Community Edition is a complete, self-contained filter. A separately
licensed commercial edition adds a flexible match engine (arbitrary
offset/length/bitmask conditions), named prefix sets for large IP lists, and
additional actions. Nothing in this repository depends on those components.

## Licence

Apache License 2.0 — see [LICENSE](LICENSE).
