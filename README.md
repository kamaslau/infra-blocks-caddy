# infra-blocks-caddy

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
![Repository size](https://img.shields.io/github/repo-size/kamaslau/infra-blocks-caddy?color=56BEB8)

## Usage

Setup DNS records for example.com (change this with the actual domain name to be resolved).

To enable Caddy to successfully acquire CA:

- Demo/Testing: Make port 80 and 443 available and disable CloudFlare DNS Delegate until CA(s) are issued successfully.
- Production: Config [CloudFlare API Token](https://dash.cloudflare.com/profile/api-tokens) for 'All zones - DNS:Edit' in Caddyfile global rules, or per-server basis with 'example.com - DNS:Edit' previledge. Follow the instructions on [Docker Hub](https://hub.docker.com/_/caddy) to learn how build an image includes CloudFlare module.

Clone and enter this repository:

```bash
git clone git@github.com:kamaslau/infra-blocks-caddy.git
cd infra-blocks-caddy
```

Ready the .env file and [Caddyfile](https://caddyserver.com/docs/caddyfile):

```bash
cp .env.example .env
nano .env

cp Caddyfile.sample Caddyfile
nano Caddyfile
```

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

## References

- [Official Documents](https://caddyserver.com/docs/)
- [Caddy on Docker Hub](https://hub.docker.com/_/caddy)
