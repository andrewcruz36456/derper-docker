# Derper

## Setup

> required: set env `DERP_DOMAIN` to your domain

```bash
docker run -e DERP_DOMAIN=derper.your-domain.com -p 80:80 -p 443:443 -p 3478:3478/udp ghcr.io/andrewcruz36456/derper
```

| env                              | required | description                                                                        | default value     |
| -------------------------------- | -------- | ---------------------------------------------------------------------------------- | ----------------- |
| DERP_DOMAIN                      | true     | derper server hostname                                                             | your-hostname.com |
| DERP_CERT_DIR                    | false    | directory to store LetsEncrypt certs (if addr's port is :443)                     | /app/certs        |
| DERP_CERT_MODE                   | false    | mode for getting a cert. possible options: manual, letsencrypt                     | letsencrypt       |
| DERP_ADDR                        | false    | listening server address                                                           | :443              |
| DERP_STUN                        | false    | also run a STUN server                                                             | true              |
| DERP_STUN_PORT                   | false    | the UDP port on which to serve STUN                                                | 3478              |
| DERP_HTTP_PORT                   | false    | the port on which to serve HTTP. Set to -1 to disable                             | 80                |
| DERP_VERIFY_CLIENTS              | false    | verify clients to this DERP server through a local tailscaled instance             | false             |
| DERP_VERIFY_CLIENT_URL           | false    | if non-empty, an admission controller URL for permitting client connections        | ""                |
| DERP_VERIFY_CLIENT_URL_FAIL_OPEN | false    | whether to fail open (allow access) if the DERP_VERIFY_CLIENT_URL is unreachable  | true              |

## Usage

Fully DERP setup official documentation: https://tailscale.com/kb/1118/custom-derp-servers/

## Client verification

In order to use `DERP_VERIFY_CLIENTS`, the container needs access to Tailscale's Local API, which can usually be accessed through `/var/run/tailscale/tailscaled.sock`. If you're running Tailscale bare-metal on Linux, adding this to the `docker run` command should be enough:

```bash
-v /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock
```

## Client verification URL

`DERP_VERIFY_CLIENT_URL` allows you to set up admission control for your DERP server via an external HTTP endpoint.

**How it works:**
1. A Tailscale client attempts to connect to your DERP server
2. The DERP server makes a POST request to your verification URL
3. Your endpoint returns `{"allow": true}` or `{"allow": false}`
4. The DERP server accepts or rejects the client based on the response

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

If the verification URL is unreachable (network timeout, server down, etc.), behavior is controlled by `DERP_VERIFY_CLIENT_URL_FAIL_OPEN`:

- `true` (default) — **allow** the connection if the verification server is unreachable. Prioritizes availability.
- `false` — **reject** the connection if the verification server is unreachable. Prioritizes security.

### Integration with Headscale

> **Note:** Requires Headscale v0.24.0 or later.

Headscale natively supports the DERP verification protocol, allowing your DERP server to verify clients directly against the Headscale node list. Point the verification URL to your Headscale instance's `/verify` endpoint:

```bash
-e DERP_VERIFY_CLIENT_URL="https://<your-headscale-domain>/verify"
```
