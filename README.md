# Derper

A lightweight Docker image for deploying a [Tailscale DERP](https://tailscale.com/kb/1118/custom-derp-servers/) relay server. The Tailscale version is automatically synced with [Headscale's go.mod](https://github.com/juanfont/headscale/blob/main/go.mod) to guarantee compatibility.

**Supported platforms:** `linux/amd64`, `linux/arm64`, `linux/arm/v7`

## Setup

> Required: set `DERP_DOMAIN` to your domain.

```bash
docker run -e DERP_DOMAIN=derper.your-domain.com \
  -p 80:80 -p 443:443 -p 3478:3478/udp \
  ghcr.io/andrewcruz36456/derper
```

### Environment variables

| env                              | required | description                                                                       | default           |
| -------------------------------- | -------- | --------------------------------------------------------------------------------- | ----------------- |
| DERP_DOMAIN                      | true     | derper server hostname                                                            | your-hostname.com |
| DERP_CERT_MODE                   | false    | certificate mode: `manual` or `letsencrypt`                                      | letsencrypt       |
| DERP_CERT_DIR                    | false    | directory for certificates                                                        | /app/certs        |
| DERP_ADDR                        | false    | HTTPS listen address                                                              | :443              |
| DERP_HTTP_PORT                   | false    | HTTP port (Let's Encrypt / health checks). Set to `-1` to disable                | 80                |
| DERP_STUN                        | false    | enable STUN server                                                                | true              |
| DERP_STUN_PORT                   | false    | UDP port for STUN                                                                 | 3478              |
| DERP_VERIFY_CLIENTS              | false    | verify clients via a local tailscaled instance                                    | false             |
| DERP_VERIFY_CLIENT_URL           | false    | admission controller URL for permitting client connections                        | ""                |
| DERP_VERIFY_CLIENT_URL_FAIL_OPEN | false    | allow access if `DERP_VERIFY_CLIENT_URL` is unreachable                          | true              |
| TZ                               | false    | timezone                                                                          | UTC               |

### Ports

| Port      | Protocol | Description        |
| --------- | -------- | ------------------ |
| 443       | TCP      | HTTPS DERP relay   |
| 80        | TCP      | HTTP / Let's Encrypt |
| 3478      | UDP      | STUN server        |

## Manual certificate mode

Mount your certificates and set `DERP_CERT_MODE=manual`:

```bash
docker run -e DERP_DOMAIN=derper.your-domain.com \
  -e DERP_CERT_MODE=manual \
  -v /etc/letsencrypt/live/derper.your-domain.com/fullchain.pem:/app/certs/derper.your-domain.com.crt:ro \
  -v /etc/letsencrypt/live/derper.your-domain.com/privkey.pem:/app/certs/derper.your-domain.com.key:ro \
  -p 80:80 -p 443:443 -p 3478:3478/udp \
  ghcr.io/andrewcruz36456/derper
```

## Health check

The container has a built-in health check against the `/derp/probe` endpoint:

```bash
docker inspect --format='{{.State.Health.Status}}' derper
```

## Usage

Full DERP setup documentation: https://tailscale.com/kb/1118/custom-derp-servers/

## Client verification

To use `DERP_VERIFY_CLIENTS`, the container needs access to Tailscale's Local API via the socket. Add this to your `docker run` command:

```bash
-v /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock
```

## Client verification URL

`DERP_VERIFY_CLIENT_URL` enables admission control via an external HTTP endpoint.

**How it works:**
1. A Tailscale client connects to your DERP server
2. The DERP server POSTs to your verification URL
3. Your endpoint returns `{"allow": true}` or `{"allow": false}`
4. The DERP server accepts or rejects the client accordingly

**Request** (sent by DERP to `DERP_VERIFY_CLIENT_URL`):
```json
{
  "nodePublic": "nodekey:abc123...",
  "source": "192.168.1.100"
}
```

**Response** (expected from your server):
```json
{
  "allow": true
}
```

### Handling verification failures

Behavior when the verification URL is unreachable is controlled by `DERP_VERIFY_CLIENT_URL_FAIL_OPEN`:

- `true` (default) — **allow** the connection. Prioritizes availability.
- `false` — **reject** the connection. Prioritizes security.

### Integration with Headscale

> **Note:** Requires Headscale v0.24.0 or later.

Headscale natively supports the DERP verification protocol. Point the verification URL to your Headscale instance:

```bash
-e DERP_VERIFY_CLIENT_URL="https://<your-headscale-domain>/verify"
```
