# Connection Transport

> Why FullCircleMUD is **WebSocket-only**, with telnet and SSH disabled — despite Evennia supporting all three natively.

---

## Overview

FullCircleMUD currently accepts player connections **only over the web client's WebSocket transport**. The Evennia framework supports telnet, SSL telnet, and SSH out of the box, but at [server/conf/settings.py:126](../src/game/server/conf/settings.py#L126):

```python
TELNET_ENABLED = False
SSH_ENABLED = False
```

This document explains why. The decision is constraint-driven, not ideological. The constraints are real but they are not permanent, and the closing section gives a checklist for revisiting it.

---

## Why Cloudflare Sits In Front of the Game

Player traffic to `fcmud.world` is proxied by Cloudflare before it reaches the Railway-hosted game container. CF's HTTP proxy provides a stack of guarantees we depend on:

### 1. `cf-ipcountry` for Compliance Tracking

Cloudflare injects an `cf-ipcountry` HTTP header on every proxied request, populated with the connecting IP's ISO-3166-1 alpha-2 country code from CF's geo-IP tables. The web client transport reads this header on the WebSocket handshake and forwards it into the Evennia session at [server/walletwebclient.py](../src/game/server/walletwebclient.py) (lines 54–58):

```python
raw_headers = getattr(self, "http_headers", {}) or {}
cf_country = raw_headers.get("cf-ipcountry")
...
server_data["geo_country"] = country
```

The country code rides into the server in the initial PCONN message so `at_post_login()` sees the correct jurisdiction *before* any player action runs. This is the load-bearing piece of our compliance posture: jurisdiction-gated features (and any future jurisdiction-specific blocks) depend on `geo_country` being authoritative for every connection. See `design/COMPLIANCE.md` § Login Geo-Logging and § Geo-Detection Re-Activation Triggers.

### 2. L3/L4 DDoS Absorption

Cloudflare's free tier absorbs volumetric L3/L4 attacks at the edge. Traffic floods never reach Railway. Without CF in front, even a small UDP/TCP flood from a single attacker can saturate a hobby-tier container.

### 3. WAF, Rate Limiting, Bot-Fight-Mode

Every HTTP/WS connection is filtered through Cloudflare's WAF rules, IP reputation scoring, and bot-fight-mode before reaching the origin. Login-attempt limits, scraper blocks, and country-level restrictions (when needed) all execute at the CF edge.

### 4. Origin IP Protection

Players resolve `fcmud.world` to a Cloudflare anycast IP. The Railway origin IP is never advertised in DNS. An attacker cannot bypass CF and target the Railway container directly because they cannot learn its address from the public DNS.

### 5. TLS Termination at the Edge

Cloudflare terminates TLS with managed certificates. The origin only sees plain HTTP from CF over a private channel. No certificate management on the game server.

---

## Why This Only Works for HTTP/WebSocket

Every protection above depends on Cloudflare *parsing the connection as HTTP*. The mechanics:

- **`cf-ipcountry` is an HTTP request header.** Cloudflare can only inject it on connections it understands as HTTP. There is no equivalent for raw TCP — the protocol has no header field to inject into.
- **WebSocket gets it because the connection begins as HTTP.** A `wss://` connection opens with an HTTP `Upgrade: websocket` request on port 443, and CF proxies that handshake. The country header lands in `self.http_headers` exactly as it would for any other proxied HTTPS request.
- **Free / Pro / Business CF plans do not proxy raw TCP at all.** Pointing an orange-cloud DNS record at a non-HTTP service simply black-holes the connection. CF expects HTTP traffic on the proxied record and drops anything else.
- **Even on Enterprise Spectrum**, CF's TCP proxy product, the byte stream is opaque. There is nowhere in a raw TCP stream for CF to inject an `ipcountry` header even if it wanted to. Spectrum gives you DDoS protection, but nothing equivalent to the header.

So *any* non-HTTP transport — telnet, telnet+TLS, SSH — leaves the CF protection envelope. There is no plan tier that fixes this; it is a property of the protocols.

---

## What Telnet/SSH Would Require to Live Alongside WebSocket

If we wanted to expose telnet (or SSH) and preserve the compliance/security posture we have today, the parallel infrastructure is roughly:

1. **Grey-cloud DNS record** for the telnet hostname, bypassing Cloudflare entirely. By doing so we accept: no DDoS protection on the telnet path, no WAF, no bot-fight-mode, and the Railway origin IP becomes publicly resolvable.
2. **Server-side IP geolocation** at session-connect time. Likely MaxMind GeoLite2 (free, weekly DB updates), looked up from `session.address` and injected into the session in the same shape `cf-ipcountry` produces. `at_post_login()` should see a unified `geo_country` field regardless of which transport the session arrived on.
3. **Railway TCP proxy verification.** Confirm whether Railway's TCP proxy preserves the original client IP (PROXY protocol or otherwise). If it does not, every telnet session arrives with Railway's internal IP as `session.address` — geo lookup is useless and IP-based abuse mitigation is impossible.
4. **A separate Railway service per port.** Railway's TCP proxy is one-port-per-service. Each exposed port (plain telnet, telnet+TLS, SSH) needs its own Railway service, and ideally its own origin IP, so exposing telnet does not leak the game/website origin.
5. **Decide on encryption.** Plain telnet sends credentials cleartext. For any public exposure we would want `TELNET_SSL_ENABLED` with a managed cert, or SSH (which carries its own host-key management), accepting that some telnet clients do not speak `telnets://`.
6. **Cleartext-credentials posture.** Even with the wallet-sign-in flow available over telnet, the password-based `connect root <pwd>` and `connect <bot> <pwd>` paths exist and would be exposed. TLS or SSH closes that.

None of these is unsolvable. They are simply work that is not free in either engineering time or operational risk, and the compliance posture during the gap is what blocks rolling them out incrementally.

---

## The Decision: WebSocket-Only

**For now, FCM is WebSocket-only.** Telnet and SSH are disabled. The website's MUD listing surfaces only the web client.

This is not an objection to telnet on principle. Telnet has been the canonical MUD transport since the 1980s; many of the genre's most loved games are telnet-first or telnet-only, and the web client is an addition to MUD culture rather than a replacement for it. **We would offer telnet if we could do it compatibly with the compliance and security posture above** — re-enabling telnet in Evennia is a one-line settings change, and the homepage already has a `{% if telnet_enabled %}` block waiting at [main-content.html:16](../src/game/web/templates/website/homepage/main-content.html#L16) that auto-lights up when the flag flips.

The blocker is the gap that opens the moment a connection is not traversing a Cloudflare HTTP proxy. The compliance/abuse posture of telnet without the parallel infrastructure listed above is unacceptable for a game that issues blockchain tokens.

**One day, when the project has more resources** — budget for MaxMind, operational headroom for a separate Railway service per transport, and engineering attention to maintain a parallel transport stack — this decision should be revisited. It is a *current* constraint, not a *permanent* architectural rule.

---

## Reactivation Checklist

A future contributor reopening telnet should work through this list rather than just flipping the flag:

1. Decide telnet's compliance posture for sessions where geo lookup returns `XX` / unknown — block, allow, or require successful lookup.
2. Implement IP geolocation on session connect, mirroring CF's `geo_country` injection so `at_post_login()` sees the same field shape regardless of transport.
3. Verify Railway TCP proxy preserves source IP (PROXY protocol) before relying on `session.address` for geo or abuse mitigation.
4. Provision a separate Railway service for the telnet port to isolate origin IP from the website service.
5. Decide on `TELNET_SSL_ENABLED` — recommended `True` for any public exposure.
6. Populate `HOSTNAME` and `PORT` in [server/conf/mssp.py](../src/game/server/conf/mssp.py).
7. Flip `TELNET_ENABLED = True` at [server/conf/settings.py:126](../src/game/server/conf/settings.py#L126).
8. Confirm the homepage telnet block at [main-content.html:16](../src/game/web/templates/website/homepage/main-content.html#L16) renders the new hostname/port correctly.
9. Test the wallet-sign-in flow end-to-end over telnet — the deeplink URL the player receives is identical to the web client, so the only difference is copy/paste UX.
