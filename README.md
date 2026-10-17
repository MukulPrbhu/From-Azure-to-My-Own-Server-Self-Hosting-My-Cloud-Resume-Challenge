# From Azure to My Own Server: Self-Hosting My Cloud Resume Challenge

I built my resume site for the [Cloud Resume Challenge](https://cloudresumechallenge.dev/) on Microsoft Azure. Then I moved it to my own hardware, running in Docker behind a Cloudflare Tunnel, and saved the $5-6 a month the Azure setup was costing me (roughly $60-70 a year) for a site that serves a few static files and one counter.

Live site: **https://www.mukulsprabhu.org**

This write-up is less a tutorial and more an honest log of the move: what I built, where I got stuck, and what each problem taught me.

## Before and after

| | Azure version | Self-hosted version |
|---|---|---|
| Static site | Azure Storage static website | nginx container |
| Counter API | Azure Function (Python) | Flask app in a container |
| Database | Azure Cosmos DB | SQLite file on a Docker volume |
| Public access | Azure CDN endpoint + Cloudflare DNS | Cloudflare Tunnel (no open ports) |
| Infrastructure | Terraform | Docker Compose |

```
Visitor --HTTPS--> Cloudflare ==encrypted tunnel==> cloudflared --HTTP--> nginx --+--> static site
                   (DNS + TLS)                      (container)          (web)    +--> /api/count --> Flask API --> SQLite
```

Three containers, one `docker-compose.yml`: `web` (nginx), `api` (Flask + gunicorn), and `cloudflared` (the tunnel connector).

## What I built

**A counter that can't lose visits.** My original function read the count from Cosmos DB, added one in Python, and wrote it back. Two simultaneous visitors could both read the same number and overwrite each other. In SQLite the increment is a single atomic statement:

```sql
UPDATE counter SET count = count + 1 WHERE id = 1 RETURNING count;
```

**A reverse proxy in front of everything.** nginx serves the static files and forwards `/api/*` to the API container. The frontend now calls a relative `/api/count` instead of a hardcoded `azurewebsites.net` URL, so there is no CORS to configure. nginx also rate-limits the anonymous counter endpoint:

```nginx
location /api/ {
    limit_req zone=counter burst=10 nodelay;
    proxy_pass http://api:8000;
}
```

**Hardened containers.** Non-root users, read-only filesystems, all Linux capabilities dropped:

```yaml
read_only: true
cap_drop: [ALL]
security_opt: ["no-new-privileges:true"]
```

**No inbound ports.** The tunnel only makes outbound connections to Cloudflare, so there is no port forwarding on my network and my home IP is never exposed.

**A counter that carried over.** The existing visitor count was passed into the new database through an environment variable on first start, so the number didn't reset to zero.

## Cloudflare settings

| Setting | Value |
|---|---|
| Tunnel type | Cloudflared, remotely managed (token-based) |
| Public hostname | `www` on `mukulsprabhu.org` |
| Service type | `HTTP` |
| Service URL | `web:8080` |
| DNS record | CNAME `www` -> `<tunnel-id>.cfargotunnel.com`, **proxied** (orange cloud) |

Two of these look odd until you know why:

- **Why HTTP, not HTTPS?** The service type describes only the last hop, from the `cloudflared` container to nginx, which happens inside a private Docker network. Visitors still connect over HTTPS, and Cloudflare terminates TLS at its edge.
- **Why `web:8080` and not `localhost`?** `cloudflared` runs in its own container, where `localhost` means itself. Containers in the same Compose project reach each other by service name, so nginx is `web`.

## Trade-offs

Self-hosting swaps a managed platform for my own responsibility. It is one machine, on home power and home internet, with no redundancy, so uptime is only as good as that machine and connection. Azure would have given me more availability guarantees for $5-6 a month. For a personal portfolio site, I'm happy with that trade, and running the thing myself was the point.
