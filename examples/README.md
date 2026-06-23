# XrayR Example Configs

This directory contains a full working example config set.  All placeholder values
are marked `CHANGE_ME` — grep and replace them for your deployment.

## How XrayR config differs from raw Xray-core

XrayR is a **panel middleware** that sits between a web panel (SSpanel, V2board,
etc.) and Xray-core.  It generates Xray-core configs dynamically — every time
the panel's user list changes, XrayR rewrites the underlying Xray-core config
and hot-reloads it.

Because of this architecture, XrayR's configuration is split into **two layers**:

| Layer | File | Purpose |
|-------|------|---------|
| XrayR control plane | `config.yml` | Panel credentials, cert management, node parameters |
| Xray-core data plane | `custom_inbound.json`, `custom_outbound.json`, `dns.json`, `route.json` | Passthrough to Xray-core (same format, with a few caveats) |

### 1. `config.yml` — this is **not** a Xray-core config

Xray-core has no `Nodes`, `ApiConfig`, `CertConfig`, or `ControllerConfig`
sections.  These are XrayR's own abstractions:

- **`Nodes`** — defines which panel(s) to pull users from and what type of
  protocol to generate for each node.  XrayR auto-generates the corresponding
  Xray-core inbounds (VMess, VLESS, Shadowsocks, etc.) from these definitions.
- **`CertConfig`** — XrayR has built-in ACME (Let's Encrypt) support.  You
  declare a DNS provider and credentials here; XrayR obtains and renews the
  certificate automatically.  In raw Xray-core you'd specify certificate file
  paths directly in each inbound's `streamSettings.tlsSettings`.
- **`ControllerConfig`** — runtime parameters like listen IP, update interval,
  device limits, and fallback rules.  These get translated into the appropriate
  Xray-core directives at generation time.

### 2. `custom_inbound.json` / `custom_outbound.json` — almost raw Xray-core

These files use the **standard Xray-core inbound/outbound format** with two
important caveats:

#### a) They are **appended**, not standalone

XrayR auto-generates inbounds/outbounds for panel users.  Your custom entries
are **appended** to those generated entries.  This means:

- The auto-generated entries come first (e.g., inbound for VMess users from
  port 443).
- Your custom entries come after (e.g., a local SOCKS proxy on 127.0.0.1:1080).
- Tags must be unique across **both** sets — if XrayR generates an inbound with
  `tag: "inbound-1"`, your custom entries cannot reuse that tag.

#### b) `sendThrough` works, `streamSettings` is inherited

- **`sendThrough`** — specifies the source IP for outbound connections from
  this outbound.  Works identically to Xray-core.
- **`streamSettings`** — the transport and TLS settings you'd normally embed in
  each Xray-core inbound/outbound work the same way here.  But note that for
  panel-managed inbounds, XrayR sets `streamSettings` automatically based on
  your panel's node configuration.

#### c) Routing glue: the `tag` field is critical

The `tag` field in these files is how `route.json` references outbounds.
Example chain:

```
custom_outbound.json:  { "tag": "ss_to_jellyfish", "protocol": "shadowsocks", ... }
route.json:            { "outboundTag": "ss_to_jellyfish", "domain": ["geosite:some-site"] }
```

The tag must match exactly — it's the bridge between your routing rules and
your protocol definitions.

### 3. `dns.json` and `route.json` — passthrough

These are **exactly** the same format as Xray-core's
[DNS](https://xtls.github.io/config/dns.html) and
[Routing](https://xtls.github.io/config/routing.html) configs.  XrayR passes
them to Xray-core unchanged.  Any Xray-core routing rule or DNS server
configuration you find in the upstream docs works here.

### Quick reference: when to use which file

| I want to... | Edit this file |
|---|---|
| Change panel API credentials | `config.yml` → `Nodes[].ApiConfig` |
| Add a new node / protocol type | `config.yml` → add a `Nodes` entry |
| Enable TLS with auto-cert | `config.yml` → `CertConfig` |
| Add a local SOCKS/HTTP proxy on the server | `custom_inbound.json` |
| Add an upstream proxy to route through | `custom_outbound.json` |
| Route specific domains through a specific outbound | `route.json` |
| Add custom DNS servers | `dns.json` |
| Block domains/IPs/protocols | `route.json` (point `outboundTag` at `block`) |

### Common pitfall: Shadowsocks outbound

⚠️  **The official Xray-core docs are inconsistent with the actual parser.**

The [upstream docs](https://xtls.github.io/config/outbounds/shadowsocks.html)
show a flat structure under `settings`:

```json
// From the docs — does NOT work in practice
{ "protocol": "shadowsocks", "settings": { "address": "1.2.3.4", "port": 8388, "method": "aes-256-gcm", ... } }
```

However the Xray-core `infra/conf` parser actually expects server parameters
wrapped in a `"servers"` array.  The flat form causes a hard panic:

```
infra/conf: 0 Shadowsocks server configured.
```

**The form that works (used in these examples):**

```json
{ "protocol": "shadowsocks", "settings": { "servers": [{ "address": "1.2.3.4", "port": 8388, ... }] } }
```

This is a Xray-core parser behavior, not XrayR-specific, but it's easy to miss
since most other outbound protocols use a flat structure.  The example files in
this directory all use the correct format.
