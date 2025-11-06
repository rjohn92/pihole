
# Pi‑hole on Home‑Server (Docker + Nginx) — README

> **Goal:** Run Pi‑hole in Docker on the home‑server (Intel NUC), expose the web UI safely behind Nginx at `https://arjayserver.duckdns.org/pihole/`, and serve DNS to the LAN from `192.168.1.154:53`.

---

## 0) What Pi‑hole actually does (and why you’ll notice it)

- **DNS sinkhole:** Pi‑hole answers DNS queries from your devices and **blocks known ad/tracker domains** by returning `0.0.0.0` instead of real IPs.
- **Network‑wide effect:** Because DNS is how *every* app finds servers, blocking at DNS removes many ads/trackers across phones, TVs, consoles, etc. (even in apps where browser ad‑blockers can’t help).
- **Observability:** You get per‑client query logs, top blocked domains, lists management, and allow/deny controls.
- **(Optional) DHCP:** Pi‑hole can hand out IPs if you want it to be the DHCP server, but using your router for DHCP is simpler and fine.

You’ll see:
- Fewer ads/tracking requests on devices.
- Query/Client graphs populating in the Pi‑hole UI.
- The ability to block entire domains, wildcard rules, or create local DNS names ( A/AAAA ).

---

## 1) Network basics (how DNS/DHCP fit here)

- **DNS (Domain Name System):** Translates names like `example.com` → IPs. Pi‑hole becomes your **LAN’s DNS server**.
- **DHCP (Dynamic Host Configuration Protocol):** Assigns IPs and *tells* devices **which DNS server to use**. We let the **router’s DHCP** hand out the Pi‑hole IP `192.168.1.154` as DNS for every device.
- **Upstream resolvers:** When Pi‑hole doesn’t know an answer, it forwards to upstream DNS (Cloudflare/Quad9 **or** local Unbound for privacy & validation).
- **VLANs (later):** Network segments (e.g., `LAN`, `IoT`, `Guests`) that can selectively use Pi‑hole and be firewalled from each other.

---

## 2) Architecture (current working state)

```
Clients ──DHCP from Router──► (get DNS = 192.168.1.154)

Clients ──DNS──► Pi‑hole (Docker @ 192.168.1.154:53)
                        │
                        └──Upstream DNS (Cloudflare/Quad9 now; Unbound later)

Browser ──HTTPS──► Nginx reverse proxy (DuckDNS)
                     ├─ /pihole/*  →  pihole/admin/  (UI)
                     └─ /api/*     →  pihole/api/    (API)
```

- Web UI: **`https://arjayserver.duckdns.org/pihole/`**
- DNS: **`192.168.1.154:53` (TCP/UDP)**

---

## 3) Docker Compose (Pi‑hole)

> **File:** `~/Containers/pihole/docker-compose.yml`

```yaml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest

    ports:
      - "192.168.1.154:53:53/tcp"
      - "192.168.1.154:53:53/udp"
      - "192.168.1.154:81:80/tcp"

    environment:
      TZ: ${Time_Zone}
      WEBPASSWORD: ${Password}                       # legacy UI login (v5)
      FTLCONF_webserver_enable_https: "false"
      FTLCONF_webserver_api_password: ${Password}    # v6 API login
      FTLCONF_dns_listeningMode: "all"

    volumes:
      - "./etc-pihole:/etc/pihole"
      - "./etc-dnsmasq.d:/etc/dnsmasq.d"

    cap_add:
      - NET_ADMIN
      - SYS_TIME
      - SYS_NICE

    restart: unless-stopped
```

**Notes**
- Ports are **pinned to the host’s LAN IP** to avoid port 53 conflicts with other bridges (e.g., libvirt’s `dnsmasq`).
- Web UI is bound at **81 → 80**; Nginx publishes it under `/pihole/` via HTTPS.

**Lifecycle**
```bash
cd ~/Containers/pihole
docker compose pull
docker compose up -d
docker logs -f pihole
```

---

## 4) Nginx Reverse Proxy (location include)

> **File:** `.../config/locations/pihole.conf`

```nginx
# Normalize bare path
location = /pihole { return 301 /pihole/; }

# API: /api/...  -> http://pihole/api/
location ^~ /api/ {
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Host  $host;
    proxy_set_header Cookie            $http_cookie;

    proxy_pass http://pihole/api/;
    proxy_http_version 1.1;
    proxy_buffering off;

    # Harden cookie flags for the 'sid' session cookie
    proxy_cookie_flags sid Secure HttpOnly SameSite=Lax;
}

# UI: /pihole/... -> http://pihole/admin/
location ^~ /pihole/ {
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Host  $host;

    proxy_pass http://pihole/admin/;
    proxy_redirect /admin/ /pihole/;

    # Rewrite absolute /admin/... in HTML so links stay under /pihole/
    proxy_set_header Accept-Encoding "";
    gzip off;
    sub_filter_once off;
    sub_filter 'href="/admin/'   'href="/pihole/';
    sub_filter 'src="/admin/'    'src="/pihole/';
    sub_filter 'action="/admin/' 'action="/pihole/';

    proxy_http_version 1.1;
    proxy_buffering off;
}

# Normalize stray /admin hits
location ^~ /admin/ { return 301 /pihole$request_uri; }
```

**Why this works**
- The **UI** actually lives at `/admin/` inside the container; we **publish it at `/pihole/`**.
- The **API** lives at `/api/` (not `/admin/api`). Mapping it 1:1 prevents login loops.
- Cookie path stays **`/`**, so the browser sends `sid` to both `/pihole` and `/api`.

---

## 5) Router setup (make the whole house use Pi‑hole)

1. In router **DHCP options**, set **DNS server = 192.168.1.154** (only this for now).
2. Renew leases or reboot a client; on that client run:
   ```bash
   nslookup google.com 192.168.1.154
   nslookup pi-hole.net 192.168.1.154
   ```
3. Open **Pi‑hole → Query Log**; you should see live queries per client.
4. In **Settings → DNS**, pick upstreams (Cloudflare/Quad9) or switch to Unbound later.

**Conditional Forwarding** (nice): set Router IP + local domain so clients show as names, not IPs.

**IPv6:** either enable Pi‑hole IPv6 and have the router hand out Pi‑hole’s v6 DNS, **or** disable IPv6 to avoid bypass via public v6 resolvers.

---

## 6) Verification checklist

- `docker logs pihole` shows **“DNS service is running”**.
- `sudo ss -tulpn | egrep ':(53|81)\s'` shows 53/udp+tcp and 81/tcp bound on `192.168.1.154`.
- `curl -i https://arjayserver.duckdns.org/api/auth` before login returns JSON `valid:false`.
- After login, browser Network tab shows `POST /api/auth -> 200` with `Set‑Cookie: sid=...; Path=/` and next `GET /api/auth -> 200`.
- Ads/telemetry requests appear as **blocked** in the Query Log.

---

## 7) Common pitfalls & fixes

- **Port 53 already in use (dnsmasq/libvirt):** pin ports to the LAN IP (as above) **or** stop the conflicting service.
- **Login loop / 401 from `/api/auth`:**
  - Ensure `/api/* → pihole/api/` (not `/admin/api`).
  - Ensure cookies are sent to `/api` (Path should be `/`). If you ever changed `proxy_cookie_path` to `/pihole/`, clear site cookies.
  - If using global SSO, **bypass it** for `/pihole/*` and `/api/*` (`auth_request off;`).
- **No queries shown:** clients still using router or public DNS → set DHCP DNS to `192.168.1.154` and renew leases.
- **IPv6 bypass:** disable IPv6 on router/clients or advertise Pi‑hole’s IPv6 as DNS; optionally block outbound DoH/DoT except from Pi‑hole.

---

## 8) Security & hardening

- Restrict DNS egress on the firewall: allow **LAN→Pi‑hole:53**, **Pi‑hole→Internet:53/443** (for upstream/updates). Block direct client DNS to the Internet (53/853/DoH) if you want to **force** Pi‑hole usage.
- Keep volumes backed up: `Settings → Teleport` to export.
- Keep images updated: `docker compose pull && docker compose up -d`.

---

## 9) Optional: add Unbound (private, validating DNS)

**Why:** cut out public resolvers, get DNSSEC validation and local cache.

**Compose service (example):**
```yaml
  unbound:
    image: mvance/unbound:latest
    container_name: unbound
    ports:
      - "127.0.0.1:5335:53/udp"
      - "127.0.0.1:5335:53/tcp"
    restart: unless-stopped
```
Then in Pi‑hole **Settings → DNS**, set **Custom** upstream: `127.0.0.1#5335` (only).

---

## 10) Optional: redundancy

- Add a second Pi‑hole (e.g., `192.168.1.155`) and set **two** DNS servers in router DHCP (`.154`, `.155`). Export/Import your lists/groups to mirror config.

---

## 11) Primer for VLANs (what we’ll do later)

- Create VLANs like **LAN (trusted)**, **IoT**, **Guest**.
- Put Pi‑hole in a **management/LAN** VLAN that others can reach only on **53/udp+tcp** and **81/tcp** (or through the Nginx proxy for the UI).
- Configure DHCP per‑VLAN to point to Pi‑hole. Use firewall rules so IoT/Guests can’t reach your NAS/servers but can reach Pi‑hole and the Internet.
- Optionally run **per‑VLAN blocking policies** with Pi‑hole Groups (e.g., stricter lists for IoT).

---

## 12) Quick commands you’ll reuse

```bash
# Container lifecycle
docker compose -f ~/Containers/pihole/docker-compose.yml up -d
docker logs -f pihole

# Nginx reload
sudo nginx -t && sudo systemctl reload nginx

# DNS sanity
nslookup google.com 192.168.1.154
dig +short doubleclick.net @192.168.1.154

# Live log
docker exec -it pihole bash -lc "tail -f /var/log/pihole/pihole.log"
```

---

## 13) Future roadmap

- Switch upstream to **Unbound** and enable **DNSSEC**.
- Add **second Pi‑hole** for HA and point DHCP to both.
- Roll out **VLANs**; apply per‑VLAN Pi‑hole Groups.
- Block **DoH/DoT** at the firewall to prevent bypass.
- Grafana/Prometheus for Pi‑hole metrics (exporter).

---

### Appendix: Why clearing cookies fixed the UI
Earlier experiments scoped the session cookie to `/pihole/`, while the API lives at `/api/`. The browser then **didn’t send the cookie to `/api`**, causing 401s. Clearing cookies forced a fresh login where Pi‑hole set `Path=/`, and the Nginx mapping left it alone — now the cookie applies to both UI and API.
