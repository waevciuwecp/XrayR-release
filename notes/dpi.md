# CMCC DPI vs. REALITY on a Budget VPS ASN — Case Study

## Summary

A VLESS + REALITY node on a budget VPS (ColoCrossing ASN) was
blocked by China Mobile's (CMCC) DPI system while remaining fully
accessible through ChinaNet (China Telecom).  The block was eventually
identified as IP-level behavioral DPI, not SNI-based, port-based, or
protocol-specific.  No free fix was found.

## Timeline

- **Day 0**: Node works on both ChinaNet and CMCC, using
  `dl.google.com` as the REALITY destination.
- **Day 1**: CMCC connections fail.  Server responds with RST to all
  non-443 ports; 443 passes TCP handshake but REALITY times out.
- **Day 2-3**: Extensive experimentation (see below).  None survives
  CMCC for more than one connection.
- **Current**: ChinaNet fully functional.  CMCC blocked at the IP
  level.  Waiting for DPI rule expiration (estimated 1-3 weeks).

## Root Cause

**Behavioral DPI on AS36352.**  ColoCrossing's IP ranges are
classified as a "VPN/proxy hosting ASN."  A fresh Let's Encrypt
certificate on a low-traffic domain from that ASN triggers the
classifier regardless of SNI, port, or protocol choice.

The "first flash success" pattern — where the very first connection
after a config change works, then all subsequent connections fail —
is textbook warm-rule DPI: the classifier takes a few packets to
profile the connection, then installs a block for the IP.

## What Was Tried

### SNI rotation
| Destination | ASN Match | Result |
|-------------|-----------|--------|
| `dl.google.com` | ❌ Google ASN | Blocked |
| `www.colocrossing.com` | ✅ AS36352 | Blocked (domain inaccessible from CMCC) |
| `www.bilibili.com` | N/A | Rejected — US IP + China domain = suspicious |
| `big-cookies.example.tld` (self-hosted) | ✅ Same IP | Blocked (cert too fresh, domain too new) |

**Conclusion**: SNI rotation alone does not bypass CMCC's DPI when the
source IP is from a flagged ASN.

### Port rotation
| Port | Result |
|------|--------|
| 443 | First connection succeeds, then blocked |
| 8443 | Blocked immediately |
| 9443 | Blocked immediately |
| 2096 | Not tested — DPI was already confirmed as IP-level |

**Conclusion**: CMCC blocks non-standard ports more aggressively than
443, but even 443 is eventually murdered.

### Self-hosted HTTPS destination

A domain was purchased, pointed at the server, and served a real HTTPS
site (nginx + Let's Encrypt) on port 443.  The REALITY inbound was
moved to a different port and configured with `dest: 127.0.0.1:443`.

**Result**: HTTPS on 443 works intermittently from CMCC (first packet
slips past the classifier).  REALITY on the non-443 port is blocked
immediately.  Co-hosting REALITY and HTTPS on 443 was the next
planned step, but CMCC blocked 443 HTTPS entirely before it could be
tested.

### TLS server implementations attempted

To serve the self-hosted HTTPS destination:

| Method | Result |
|--------|--------|
| Python `http.server` + `ssl.wrap_socket` on listening socket | Recv-Q fills, never accepts |
| Python `http.server` + per-connection `wrap_socket` via `get_request` | Hangs on first request |
| Python raw sockets + `ssl.SSLContext` | Hangs on TLS handshake |
| `openssl s_server` | Doesn't respond to HTTP after TLS |
| `socat openssl-listen` | Quoting issues, SSL errors from REALITY handshakes |
| **nginx** | Worked immediately. 403 fixed with `chmod 755 /root`. |

**Lesson**: Use nginx.  Every TLS-accepting server that is not
nginx is not worth the fight.

### Protocol alternatives considered

| Option | Why Rejected |
|--------|-------------|
| Hysteria2 (QUIC) | Previous VPS got IP-permanently-blocked by CMCC after running HY2 |
| Cloudflare CDN + WS/TLS | 100-second connection limit kills downloads/long sessions |
| Cloudflare Tunnel | Incompatible with REALITY (CDN terminates TLS, REALITY needs raw TCP) |
| Oracle Free Tier relay | Requires credit card for signup |
| BuyVM $3.50/mo relay | Requires payment method |
| HK VPS inbound | $10-15/mo, too expensive |
| IPv6 | VPS had no IPv6 address |
| Port 80 | Can't serve TLS on 80 |

## What Actually Works

| Carrier | Status |
|---------|--------|
| ChinaNet (Telecom) WiFi/Broadband | ✅ Fully functional |
| ChinaNet (Telecom) Cellular | Probably fine, untested |
| CMCC (Mobile) | ❌ IP-level DPI block |
| CU (Unicom) | Untested |

## The Free Fix Roadmap

1. **Wait 1-3 weeks.**  CMCC DPI rules expire if no suspicious traffic
   persists.  Test periodically on cellular.
2. **If it lifts**: keep traffic patterns low-key.  Don't serve TLS
   on the same IP as REALITY.
3. **If it doesn't lift**: cheapest fix is a $3.50/mo BuyVM relay or
   a friend's IP.  Cloudflare Tunnel + VLESS+WS+TLS for browsing only
   (accepts the 100s timeout).
4. **If the whole ASN is burned**: switch providers.  Clean-ASN
   options at $3-6/mo include BuyVM, Vultr, and AWS Lightsail.

## General Lessons

- **Budget VPS ASNs are flagged.**  ColoCrossing, RackNerd, and
  similar providers are classified as "hosting/VPN" by Chinese
  carriers.  REALITY's SNI masking cannot overcome IP reputation.
- **The REALITY destination must share the server's ASN** for the
  SNI-to-IP correlation to pass DPI.  This is almost impossible on
  budget VPS ASNs because no popular domain has CDN edges on them.
- **A self-hosted domain with a real cert** is the correct answer,
  but if the IP is already flagged, it won't save you.
- **ChinaNet is less aggressive than CMCC.**  Different carriers
  deploy different DPI stacks with different rule sets.
- **Use nginx.**  Every other minimal TLS server is a waste of time.
- **The "first flash success" pattern** is a reliable indicator of
  warm-rule DPI.  If a new config works exactly once and then fails,
  you're dealing with active classification.


