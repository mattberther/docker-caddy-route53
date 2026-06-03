# docker-caddy-route53

A minimal [Caddy](https://caddyserver.com/) 2.11 image with the [`caddy-dns/route53`](https://github.com/caddy-dns/route53) DNS provider module baked in, so Caddy can solve ACME **DNS-01 challenges** using Amazon Route 53.

The stock `caddy` image doesn't ship DNS provider modules, so this repo uses `xcaddy` to compile a custom binary and drops it into the official runtime image.

## Why build this?

The DNS-01 challenge proves domain control by creating a TXT record instead of serving a file over HTTP. That lets you:

- Issue **wildcard certificates** (`*.example.com`), which HTTP-01 can't do.
- Obtain certificates for hosts that **aren't publicly reachable** on ports 80/443
  (internal services, split-horizon DNS, behind a firewall, etc.).

## What's in the image

| Component | Version |
| --- | --- |
| Caddy | `2.11` |
| `caddy-dns/route53` | `v1.6.2` |

> **Compatibility note:** Caddy 2.10+ moved to libdns 1.0, which breaks older DNS provider versions. The pinned `route53@v1.6.2` is compatible with the `caddy:2.11` base used here. If you bump the Caddy base image, check that the module version still matches.

## Build

```bash
docker build -t caddy-route53 .
```

The `Dockerfile` is a two-stage build: the `caddy:2.11-builder` stage compiles Caddy with the Route 53 module via `xcaddy`, and the final stage copies the resulting binary into the slim `caddy:2.11` runtime image.

## Usage

### 1. Provide AWS credentials

The module reads from the standard AWS SDK credential chain, so the cleanest approach in containers is environment variables (it also honors `~/.aws/credentials`, EC2 instance roles, and IRSA):

| Variable | Notes |
| --- | --- |
| `AWS_ACCESS_KEY_ID` | Required unless using an instance/role profile |
| `AWS_SECRET_ACCESS_KEY` | Required unless using an instance/role profile |
| `AWS_REGION` | Optional; defaults to `us-east-1` |

The IAM principal needs permission to read your hosted zones and write TXT records. A minimal policy (scope the `hostedzone` resource to your zone ID):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["route53:ListHostedZonesByName", "route53:ListHostedZones"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["route53:GetChange"],
      "Resource": "arn:aws:route53:::change/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:ListResourceRecordSets"
      ],
      "Resource": "arn:aws:route53:::hostedzone/ZONE_ID_HERE"
    }
  ]
}
```

### 2. Configure the `tls` directive

In your `Caddyfile`, point the `tls` directive at the `route53` DNS provider.  With credentials supplied via environment variables, this is all you need:

```caddyfile
*.example.com, example.com {
    tls {
        dns route53 {
            region us-east-1
        }
    }
    respond "Hello from Caddy + Route 53"
}
```

You can also set credentials and tuning options explicitly in the block instead of (or in addition to) the environment. The supported options for v1.6.x:

```caddyfile
tls {
    dns route53 {
        access_key_id     "AKI..."   # or $AWS_ACCESS_KEY_ID
        secret_access_key "wJa..."   # or $AWS_SECRET_ACCESS_KEY
        region            "us-east-1"
        profile           "real-profile"   # optional named profile
        max_retries       5                # optional
        wait_for_route53_sync false        # optional; default false since 1.6.2
    }
}
```

### 3. Run

```bash
docker run -d \
  --name caddy \
  -p 80:80 -p 443:443 -p 443:443/udp \
  -e AWS_ACCESS_KEY_ID="AKI..." \
  -e AWS_SECRET_ACCESS_KEY="wJa..." \
  -e AWS_REGION="us-east-1" \
  -v "$PWD/Caddyfile:/etc/caddy/Caddyfile" \
  -v caddy_data:/data \
  -v caddy_config:/config \
  caddy-route53
```

The `caddy_data` volume is important — it persists issued certificates and ACME account keys across container restarts so you don't re-request certs (and risk hitting rate limits) every time.

### docker compose

```yaml
services:
  caddy:
    build: .
    # Or use a prebuilt image instead of building locally:
    # image: caddy-route53
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"   # HTTP/3
    environment:
      AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
      AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"
      AWS_REGION: "us-east-1"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config

volumes:
  caddy_data:
  caddy_config:
```

## Tips

- To test without burning Let's Encrypt rate limits, point Caddy at the staging CA with a global option:
  ```caddyfile
  {
      acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
  }
  ```
- Verify the module is actually compiled into the binary:
  ```bash
  docker run --rm caddy-route53 caddy list-modules | grep route53
  # dns.providers.route53
  ```

## License

Caddy and the `caddy-dns/route53` module are distributed under their respective licenses (both Apache-2.0). This repository only provides the build recipe.
