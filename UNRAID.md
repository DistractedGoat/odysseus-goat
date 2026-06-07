# Running Odysseus safely on Unraid

This is a deployment guide for an **internal/home** Odysseus install on Unraid,
with the security-relevant settings spelled out. It assumes the
Docker-Compose-Manager plugin (or any `docker compose` workflow).

> TL;DR: the defaults are safe (loopback-bound, auth on). The only thing that
> makes it network-reachable is **you** changing the bind address. Do that
> deliberately, keep auth on, and put HTTPS in front before any remote access.

---

## 1. First boot (minimal, safe)

1. Clone into your appdata share, e.g. `/mnt/user/appdata/odysseus`.
2. Copy `.env.unraid.example` to `.env` and review it.
3. `docker compose up -d --build`
4. Get the generated admin password:
   `docker compose logs odysseus | grep -i password`
5. Open `http://<unraid-ip>:7000`, log in, **change the password immediately**,
   and enable **2FA** (Settings → Security).

Bundled services (ChromaDB `8100`, SearXNG `8080`, ntfy `8091`) are bound to
`127.0.0.1` and should **stay** internal-only. Don't publish them.

---

## 2. Security checklist (do all of these)

- [ ] `AUTH_ENABLED=true` and `LOCALHOST_BYPASS=false` (the defaults — keep them).
- [ ] Strong admin password + 2FA. Don't enable open signup.
- [ ] **Never port-forward `7000` to the internet.** For remote access use
      Tailscale or an HTTPS reverse proxy (below).
- [ ] Leave ChromaDB/SearXNG/ntfy on loopback.
- [ ] `PUID=99 PGID=100` so bind-mounted `data/`/`logs/` stay editable on Unraid.
- [ ] You are the only **admin**. Admin = shell / file / model-serving / MCP
      access by design; non-admins are blocked from those.

A note on the agent: its shell and Cookbook tools run with **no network-egress
sandbox** (an upstream-acknowledged limitation), so an admin-driven agent can
reach your LAN. That's fine for personal internal use — just don't hand admin
to anyone you don't trust.

---

## 3. What "HTTPS reverse proxy" means (and why)

Odysseus speaks **plain HTTP** on `7000` — traffic, including your login cookie,
is unencrypted. On a trusted LAN that's tolerable; for anything beyond it, put a
**reverse proxy** in front:

```
browser ──HTTPS (encrypted, real cert)──▶ reverse proxy ──plain HTTP──▶ Odysseus:7000
                                          (on Unraid)      (same box, never leaves it)
```

The proxy terminates TLS (so the browser sees a valid certificate and a lock
icon) and forwards to Odysseus locally. It must send `X-Forwarded-Proto: https`
— Odysseus reads that header to mark the session cookie `Secure`. NPM, Caddy,
Traefik, and Tailscale Serve all do this automatically.

When you front it with HTTPS, also:
- set `SECURE_COOKIES=true`, and
- add your HTTPS origin to `ALLOWED_ORIGINS`
  (e.g. `https://odysseus.example.com`).

---

## 4. Three ways to do it (easiest first)

### Option A — Tailscale (recommended for "just me", no certs, no port-forward)

Tailscale is a private WireGuard network between your devices. Nothing is
exposed to the public internet, and **Tailscale Serve** gives you a real HTTPS
certificate automatically.

1. Install the Tailscale plugin/container on Unraid; `tailscale up`.
2. Keep Odysseus published only where Tailscale can reach it (LAN bind is fine).
3. Run on the Unraid host:
   ```bash
   tailscale serve --bg --https=443 http://127.0.0.1:7000
   ```
4. Open `https://<your-unraid-name>.<tailnet>.ts.net` from any device on your
   tailnet. Real cert, encrypted, no public exposure.
5. In `.env`: `SECURE_COOKIES=true` and add that URL to `ALLOWED_ORIGINS`.

### Option B — Nginx Proxy Manager (GUI, good for a real hostname)

1. Install **Nginx Proxy Manager** from Community Apps.
2. Put NPM and Odysseus on the **same custom Docker network** (e.g. `proxynet`)
   so NPM can reach Odysseus by container name. Ideally then **don't publish**
   Odysseus's `7000` to the host at all, so it's reachable *only* through NPM.
3. NPM → Hosts → Proxy Hosts → Add:
   - Domain: `odysseus.yourdomain.com`
   - Scheme `http`, Forward Hostname `odysseus`, Forward Port `7000`
   - Enable **Websockets Support** (chat streaming) and **Block Common Exploits**
   - SSL tab: request a **Let's Encrypt** certificate, force SSL, enable HTTP/2
4. In `.env`: `SECURE_COOKIES=true`, add `https://odysseus.yourdomain.com` to
   `ALLOWED_ORIGINS`.

> Trade-off: if you keep `7000` published on the LAN *and* proxy in front, the
> plain-HTTP port is still directly reachable on your LAN (auth still required).
> The "same Docker network, don't publish 7000" approach above closes that.

### Option C — Caddy (simplest config file, automatic HTTPS)

A whole Caddy config is two lines. Put Caddy and Odysseus on the same Docker
network; `Caddyfile`:

```caddyfile
odysseus.yourdomain.com {
    reverse_proxy odysseus:7000
}
```

Caddy auto-provisions and renews a Let's Encrypt certificate (needs public DNS
pointing at you + ports 80/443 reachable). For **LAN-only HTTPS without a public
domain**, use Caddy's internal CA:

```caddyfile
odysseus.home {
    tls internal
    reverse_proxy odysseus:7000
}
```

(then trust Caddy's local root CA on your devices). In `.env`:
`SECURE_COOKIES=true` and add the `https://…` origin to `ALLOWED_ORIGINS`.

---

## 5. Email client — safe to use

The email client (read / compose / send / manual reply) is safe: credentials
are encrypted at rest, TLS is verified, mailboxes are owner-isolated, and
incoming HTML is sanitized.

The AI triage features (auto-summarize / auto-reply / auto-tag / auto-spam /
auto-calendar) are **off by default**. They process untrusted email through the
LLM; this fork wraps that content as untrusted data to blunt prompt injection.
Recommended: auto-summarize / auto-reply are fine (drafts are always
user-reviewed). Leave **auto-calendar off** — it's the one path that can write
to your calendar from an incoming email. Avoid driving email through the *agent*
with tools enabled.

---

## 6. Don't

- Don't expose `7000` (or ChromaDB/SearXNG/ntfy) directly to the internet.
- Don't set `AUTH_ENABLED=false` or `LOCALHOST_BYPASS=true` on a networked box.
- Don't run with the generated admin password — change it on first login.
- Don't enable open signup unless you intend to.
