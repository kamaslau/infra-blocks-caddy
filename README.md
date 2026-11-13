# infra-blocks-caddy

[![Blocks](https://img.shields.io/badge/infra--blocks-index-555?style=flat-square)](https://github.com/kamaslau/infra-blocks)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
![Repository size](https://img.shields.io/github/repo-size/kamaslau/infra-blocks-caddy?color=56BEB8)

A unit, not a demo: one service, one port, one documented credential, disposable with a single command.

Typical use: the client's port 80 is closed by policy, or the domain already sits behind Cloudflare, and something still has to terminate TLS and route the traffic.

[Caddy](https://caddyserver.com/docs/) as a reverse proxy, with certificates issued over DNS-01 through Cloudflare instead of the HTTP challenge.

## Service URL

- HTTPS `https://localhost:443`
- HTTP `http://localhost:80` — every site block that has TLS redirects here
- HTTP/3 `443/udp`

There are no port variables in `.env`: the container runs with `network_mode: host`, so it binds the host's own 80 and 443 and a mapping would be a second layer that only hides the first. Moving a port means editing the `Caddyfile` and the host firewall.

## Default user

None. Caddy has no login; what it needs is a hostname that resolves to this machine and, for DNS-01, a Cloudflare token.

| Variable | Meaning |
| -------- | ------- |
| `CA_EMAIL` | Contact address for the ACME account |
| `CLOUDFLARE_API_TOKEN` | Token with `Zone → DNS → Edit` on the zone being issued for |

Defaults are empty, and an empty `CLOUDFLARE_API_TOKEN` is why a certificate request fails with `no solver` rather than a DNS error.

## Usage

Setup DNS records for `example.com` — replace it with the domain actually being served. Numbers to get right before the first `up`:

- **Production**: a Cloudflare API token with `All zones - DNS:Edit`, or per-zone `example.com - DNS:Edit`. Without the DNS provider module, DNS-01 cannot run; the `Dockerfile` in this repository builds an image that has it.
- **Demo or testing**: port 80 and 443 reachable from the internet, and Cloudflare's DNS proxy (the orange cloud) switched off until the certificate is issued. The HTTP challenge fails silently behind the proxy, and the failure looks like a timeout.

Clone and enter this repository:

```bash
git clone git@github.com:kamaslau/infra-blocks-caddy.git
cd infra-blocks-caddy
```

Ready the `.env` file and the [Caddyfile](https://caddyserver.com/docs/caddyfile):

```bash
cp .env.example .env
nano .env

cp Caddyfile.sample Caddyfile
nano Caddyfile
```

`Caddyfile` is gitignored on purpose: it holds the client's hostnames. `Caddyfile.sample` keeps a `localhost` block that serves `sites/` and a `www.example.com` block that proxies to `:3000`, so both halves of the job are visible before you edit anything.

Start up with [Docker Compose](https://docs.docker.com/compose/):

```bash
docker compose up -d
```

Update existing composed containers using latest images:

```bash
docker compose pull && \
docker compose down && \
docker compose up -d
```

### Building the image with the Cloudflare module

`image: caddy` in `compose.yml` is the stock image and cannot solve DNS-01. To use the `dns cloudflare` directive, either build the image in this repository — uncomment the `build:` block in `compose.yml` — or pull the one the workflow publishes to GHCR on every push to `main`.

## Verify

```bash
docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile
```

`Valid configuration` means the Caddyfile parses and every directive is backed by a module in this binary — including `dns cloudflare`, which is the check the stock image fails.

```bash
curl -sI http://localhost | head -1        # HTTP/1.1 200 OK       — sites/index.html
curl -sI https://localhost -k | head -1    # HTTP/2 200             — local certificate
```

The local certificate is issued by Caddy's internal CA, so `-k` is expected; a browser warning on `localhost` is not a fault. For a real domain, `curl -sI https://example.com | head -1` returning `200` is the proof that DNS-01 completed, and `docker compose logs caddy | grep -i certificate` says which challenge was used.

## Why this shape

`network_mode: host` rather than port mappings, because Caddy's job here is to own ports 80 and 443 of the machine and terminate TLS on them. Bridge networking with published ports works for a demo and then breaks on HTTP/3, on client IPs (everything arrives from the bridge gateway) and on the redirect from `http://` to `https://` that the client tests from outside.

DNS-01 rather than the HTTP challenge, because the unit exists for hosts where port 80 is closed. DNS-01 also survives a renewal during a firewall change, which is when a certificate usually expires.

The certificate and account data live in the `caddy_data` volume. Losing it means re-issuing everything on the next `up`, and Let's Encrypt rate-limits duplicate certificates per domain per week — the volume is the difference between a restart and a week of waiting.

`admin off` in the sample global block disables Caddy's admin API. It is on `localhost:2019` by default, which is a local privilege worth removing on a shared host; re-enable it while debugging, not afterwards.

## Notes for deployments in China

The stock `caddy` image pulls from Docker Hub and needs a registry mirror. The DNS-01 path is the reason this unit travels well: ACME over port 443 to Let's Encrypt or ZeroSSL works from most mainland hosts, while the HTTP challenge dies at a closed port 80.

If the site is served to mainland users through Cloudflare, keep the proxy on after issuance and set the origin to a hostname that only Caddy answers, so the origin certificate and the edge certificate are separate concerns.

---

Part of [infrastructure blocks](https://github.com/kamaslau/infra-blocks) · ← back to index
