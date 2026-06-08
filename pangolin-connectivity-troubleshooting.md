Pangolin & Pelican/Wings Connectivity Troubleshooting
What I Did
Set up and debugged a self-hosted infrastructure stack connecting public HTTPS subdomains to internal services through Pangolin, Traefik, and Pelican/Wings—without exposing backend ports directly to the internet.

This wasn't just "get it working." It was about understanding why things failed at each layer and developing repeatable patterns I could use going forward.

The Setup
Internet → DNS → Traefik/Nginx (HTTPS termination) → Internal LAN → Backend Services
                ↓                              ↓
            Pangolin Dashboard           Pelican Panel + Wings Node
Key components:

Custom domain DNS routing
Let's Encrypt/TLS certificate management
Traefik reverse proxy for ingress
Pelican Panel (control UI) + Wings (node API)
WebSocket support for live console connections
Problems I Solved
1. Service Works Internally but Fails Publicly
Symptom: The panel loaded, browser functionality broke. Backend responded fine over LAN. What I learned: Don't jump straight to DNS/proxy issues. Test the backend directly first. Once confirmed, move up the stack methodically.

2. WebSocket Console Would Drop
Symptom: Panel loaded, but wss:// console connections timed out. Root cause: Mismatch between what the panel thought its public URL was and what Traefik was actually receiving. Also missing/incorrect X-Forwarded-Proto headers. Fix: Aligned APP_URL, node FQDN, and proxy config to all use the same external hostname with proper scheme handling.

3. Certificate Mismatch Caused HTTPS Failures
Symptom: Web server wouldn't start or TLS handshakes failed. Why: Small typos—like service.example.com vs svc-example.com—were scattered across DNS records, Nginx configs, Certbot paths, .env files, and browser URLs. Approach: Audited every single place the hostname appeared. Nothing can mismatch, even by one character.

4. Port 80 Was Already Taken
Symptom: Nginx wouldn't bind because Apache had grabbed port 80 first. Tool used: sudo ss -tulpn | grep ':80' Resolution: Chose a single web listener and made sure others ran behind it instead of competing for the same ports.

5. Database Authentication Errors
Symptom: Pelican reported "Access denied for user 'pelican'@'localhost'" Investigation: Credentials in .env didn't match what was actually granted in MySQL/MariaDB. Lesson: Network routing can be perfect while auth fails. Always verify service accounts independently.

6. Panel vs. Node Hostname Confusion
Issue: During testing, roles got swapped between hostnames. Debugging became confusing. Solution: Documented a role-based naming convention:

panel.example.com → Pelican UI
node.example.com → Wings API + WebSocket target
pang.example.com → Pangolin dashboard
How I Approached Troubleshooting
Built a layered checklist I actually use when routes fail:

Layer	Check
DNS	Does nslookup/dig return the right endpoint? No cached/wildcard interference?
HTTPS	Does curl -I get a valid response? Certificate matches the hostname?
Proxy	Do Traefik logs show the request hitting the right router? Routes configured correctly?
Backend	Can you reach it directly via IP/port? Is it bound to 0.0.0.0 not just 127.0.0.1?
App Config	Public URL, DB credentials, node FQDN all consistent?
What This Taught Me
Test from the inside out. You're guessing until you confirm the backend is actually alive.
WebSockets are pickier than HTTP. A working page load doesn't guarantee wss:// will work—upgrade headers and scheme consistency matter more here.
Documentation prevents back-and-forth. Clear hostname ownership saved hours when configurations shifted during testing.
One thing per port. If two services fight for port 80, neither works cleanly. Pick your ingress layer early.
Stack Summary
Linux, Docker, Traefik, Nginx, Certbot, MySQL/MariaDB, Pelican, Wings, Pangolin, DNS providers, curl, ss, lsof, journalctl
