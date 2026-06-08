# Pangolin, Domain Routing, and Pelican/Wings Connectivity Troubleshooting

## Project Context

This document summarizes troubleshooting performed while connecting self-hosted services through Pangolin, Traefik, custom domain DNS, and internal Linux VMs. The main focus was validating that external HTTPS subdomains reached the correct internal services without exposing backend ports directly.

The most important troubleshooting pattern was to separate the problem into layers:

1. DNS resolution
2. Public ingress / HTTPS
3. Pangolin resource mapping
4. Traefik reverse proxy behavior
5. Backend service availability
6. Application-level configuration
7. WebSocket behavior for console connections

---

## Issue 1: Backend Service Was Reachable Internally but Public Route Still Failed

### Symptom

A public service hostname reached the expected backend behavior, but browser-based functionality still failed. In the Pelican/Wings setup, the node/Wings service responded over plain HTTP on the LAN and returned a JSON authorization-style response, confirming the service was alive.

Portfolio-safe example:

```text
Internal test:
curl http://192.168.x.x:8080/

Public test:
curl https://node.example.com/
```

If both return the same application-level response, the backend is probably reachable and the problem is no longer basic service availability.

### Investigation

Checked whether the backend service was bound to the correct interface and port. The service was configured to listen on all interfaces and port `8080`, which meant LAN clients and the reverse proxy could reach it.

Example pattern:

```yaml
host: 0.0.0.0
port: 8080
```

### Resolution / Lesson

Once local backend reachability was confirmed, troubleshooting moved upward to DNS, proxy routes, TLS, application URL settings, and WebSocket handling.

### What this proves

This demonstrates layered troubleshooting: do not assume the public hostname is the root cause until the internal service path is tested directly.

---

## Issue 2: WebSocket Console Failed Through the Proxy

### Symptom

The Pelican console attempted to connect using a secure WebSocket URL similar to:

```text
wss://node.example.com/api/servers/<server-id>/ws
```

The panel loaded, but the live console connection failed.

### Likely Causes

A WebSocket issue in this setup can happen when:

- The panel is configured with the wrong public URL.
- The node/Wings public FQDN does not match the expected route.
- The reverse proxy does not pass WebSocket upgrade headers correctly.
- HTTPS terminates at the proxy but the app believes it is serving plain HTTP.
- `X-Forwarded-Proto` or equivalent forwarding headers are missing or incorrect.
- The proxy routes the main HTTP request correctly but mishandles `wss://` traffic.

### Troubleshooting Steps

1. Confirm the panel URL is HTTPS in the application environment config.

```text
APP_URL=https://panel.example.com
```

2. Confirm the node/Wings FQDN is the public hostname expected by the panel.

```text
Node FQDN: https://node.example.com
```

3. Confirm the backend Wings service is reachable privately.

```bash
curl http://192.168.x.x:8080/
```

4. Confirm the public node hostname reaches the same backend behavior.

```bash
curl https://node.example.com/
```

5. Check proxy logs during a console connection attempt.

```bash
docker logs <traefik-container-name>
journalctl -u pelican-wings.service -f
```

6. Verify WebSocket support through the reverse proxy route.

Important proxy behavior:

```text
Client uses:     wss://node.example.com
Proxy receives:  HTTPS / WebSocket upgrade
Backend uses:    http://192.168.x.x:8080
```

### Resolution / Lesson

The panel, public node FQDN, and reverse proxy route all need to agree on the same external HTTPS names. A backend service can be healthy while WebSocket console access still fails because WebSocket routing is more sensitive to scheme, hostname, and upgrade handling than a normal page load.

### What this proves

This is strong portfolio material because WebSocket failures are common in real infrastructure work. The fix requires understanding HTTP vs HTTPS, WebSocket upgrade behavior, reverse proxies, and application URL configuration.

---

## Issue 3: Domain / Certificate Mismatch Broke HTTPS

### Symptom

The web server failed to start or HTTPS failed because the configured domain name did not match the certificate path or expected hostname.

A common version of this issue looks like:

```text
Nginx references:       service.example.com
Certificate path uses:  different-service-or-misspelled-domain.example.com
DNS record uses:        another hostname
```

### Investigation

Checked these locations for exact domain consistency:

- Public DNS provider
- Nginx `server_name`
- Certbot / Let's Encrypt certificate path
- Pangolin resource hostname
- Application `.env` / public URL
- Browser URL
- Panel node FQDN

### Resolution / Lesson

All domain references must match exactly. Even a small typo between similar domain strings can break TLS, route traffic to the wrong service, or cause the app to generate incorrect URLs.

### What this proves

This demonstrates practical TLS and DNS troubleshooting. The issue was not solved by guessing; it was solved by comparing every layer that referenced the hostname.

---

## Issue 4: Port 80 Conflict Blocked the Web Server

### Symptom

Nginx could not bind to port `80` because another service was already listening on the same port. In the setup history, Apache was detected occupying port `80`.

### Investigation Commands

```bash
sudo ss -tulpn | grep ':80'
sudo lsof -i :80
```

Example result pattern:

```text
apache2 is listening on 0.0.0.0:80
```

### Resolution Options

Use only one service as the public web listener on port `80` / `443`, or move the conflicting service behind the reverse proxy.

Possible fixes:

```bash
sudo systemctl stop apache2
sudo systemctl disable apache2
sudo systemctl restart nginx
```

or:

```text
Move Apache/Nginx application behind Pangolin/Traefik instead of binding directly to the public ports.
```

### What this proves

This demonstrates basic but important Linux network troubleshooting: checking listening ports, understanding bind conflicts, and keeping clear ownership of ingress ports.

---

## Issue 5: Application Database Login Failed

### Symptom

Pelican returned a database authentication error similar to:

```text
ERROR 1045 (28000): Access denied for user 'pelican'@'localhost' (using password: YES)
```

### Investigation

Checked whether the database username, password, database name, and host in the application environment file matched the actual MySQL/MariaDB user grants.

Typical config file location:

```text
/var/www/pelican/.env
```

Database validation commands:

```bash
mysql -u pelican -p
mysql -u root -p
```

### Resolution / Lesson

The service account and the application `.env` file must agree. Network routing can be correct while the application still fails because the backend database credentials are wrong.

### What this proves

This shows full-stack infrastructure troubleshooting across network, web server, application, and database layers.

---

## Issue 6: Confusion Between Panel and Node Hostnames

### Symptom

The panel UI and node/Wings service used different subdomains. At different points in the setup, the panel and node roles were moved between hostnames during testing.

A safe final naming pattern is:

```text
panel.example.com  -> Pelican Panel UI
node.example.com   -> Wings/node API and WebSocket target
pang.example.com   -> Pangolin dashboard
```

### Investigation

Mapped each hostname to its actual role:

| Hostname role | Expected behavior |
|---|---|
| Pangolin dashboard | Loads the Pangolin UI/auth portal |
| Panel UI | Loads the Pelican web interface |
| Node/Wings FQDN | Returns node/Wings API behavior and supports console WebSockets |

### Resolution / Lesson

The final documentation should use role-based naming. This avoids confusion when the actual subdomain names change during migration or testing.

### What this proves

This demonstrates infrastructure documentation discipline: hostname ownership should be clear, consistent, and tied to service purpose.

---

## Troubleshooting Checklist

Use this checklist when a public service route fails.

### DNS Layer

```bash
nslookup panel.example.com
dig panel.example.com
```

Confirm:

- Hostname resolves.
- The result points to the expected ingress endpoint.
- Old records are not cached.
- Wildcard records do not accidentally route the wrong service.

### Public HTTPS Layer

```bash
curl -I https://panel.example.com
```

Confirm:

- HTTPS responds.
- Certificate matches the hostname.
- Redirects do not point to an old or wrong domain.

### Pangolin / Traefik Layer

Confirm:

- Resource exists.
- Resource hostname matches DNS.
- Resource target points to the correct backend protocol, IP, and port.
- Access policy allows the intended user.
- Traefik logs show the request reaching the expected router.

### Backend Service Layer

```bash
curl http://192.168.x.x:8080/
```

Confirm:

- Backend service is listening.
- Backend responds locally.
- Linux firewall allows proxy-to-backend traffic.
- Service is bound to `0.0.0.0` or the correct interface, not only `127.0.0.1`.

### Application Layer

Confirm:

- Public app URL is correct.
- Database credentials are correct.
- Node FQDN matches the panel config.
- WebSocket URL uses `wss://` when the public site is HTTPS.

---

## Portfolio Summary

Troubleshot and documented a multi-layer remote-access deployment using Pangolin, Traefik, custom DNS, HTTPS routing, Pelican Panel, and Wings. Diagnosed issues across DNS resolution, TLS certificate matching, proxy routing, WebSocket console connections, Linux port conflicts, backend service binding, and database authentication. Used direct internal service testing and public hostname validation to isolate whether failures were caused by DNS, proxy configuration, application settings, or backend services.
