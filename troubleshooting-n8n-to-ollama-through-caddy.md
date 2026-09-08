# Connecting n8n to Ollama through Caddy

This guide documents how to connect a Docker-based n8n installation to a local Ollama server exposed through Caddy at an HTTPS address such as:

```text
https://ollama.home
```

The final working route is:

```text
n8n container → https://ollama.home/v1 → Caddy → http://127.0.0.1:11434 → Ollama
```

## Prerequisites

- Ollama is running on port `11434`.
- Caddy is installed on the host rather than inside Docker.
- n8n is running in Docker.
- Local DNS resolves `ollama.home` to the server running Caddy.
- Client devices already trust Caddy's local certificate authority where required.

## 1. Configure Caddy

Add the following site to `/etc/caddy/Caddyfile`:

```caddyfile
ollama.home {
    tls internal
    reverse_proxy 127.0.0.1:11434
}
```

Validate and reload Caddy:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

Test from the host:

```bash
curl -k https://ollama.home/api/tags
```

A successful response contains a JSON list of installed Ollama models.

## 2. The initial n8n API-key error

When configuring n8n's **Self-hosted or OpenAI-compatible endpoint**, the initial test may fail with:

```text
OpenAI API key is missing. Pass it using the 'apiKey' parameter or the
OPENAI_API_KEY environment variable.
```

Ollama itself does not require an API key. However, n8n uses an OpenAI-compatible client for this connection, and that client requires the API-key field to contain a value.

Enter a harmless placeholder:

```text
ollama
```

Ollama ignores this value.

## 3. The certificate trust error

Testing the Caddy endpoint from inside the n8n container may produce:

```text
unable to get local issuer certificate
```

or:

```text
wget: error getting response: Connection reset by peer
```

This happens because the host and browser may trust Caddy's private certificate authority, but Docker containers have their own certificate environment. The n8n container does not automatically inherit certificates trusted by the host.

### Copy Caddy's public root certificate

From the directory containing the n8n Compose file:

```bash
sudo install -m 0644 \
  /var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt \
  ./caddy-root.crt
```

Only the public root certificate is copied. Do not copy Caddy's `root.key` file.

### Mount the certificate into n8n

Use the following `compose.yaml`:

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped

    ports:
      - "127.0.0.1:5678:5678"

    environment:
      TZ: Europe/London
      GENERIC_TIMEZONE: Europe/London

      N8N_HOST: n8n.home
      N8N_PORT: "5678"
      N8N_PROTOCOL: https
      N8N_EDITOR_BASE_URL: https://n8n.home/
      WEBHOOK_URL: https://n8n.home/
      N8N_PROXY_HOPS: "1"

      N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS: "true"
      N8N_RUNNERS_ENABLED: "true"

      NODE_EXTRA_CA_CERTS: /etc/ssl/certs/caddy-root.crt

    volumes:
      - n8n_data:/home/node/.n8n
      - ./caddy-root.crt:/etc/ssl/certs/caddy-root.crt:ro

volumes:
  n8n_data:
```

The important additions are:

```yaml
environment:
  NODE_EXTRA_CA_CERTS: /etc/ssl/certs/caddy-root.crt

volumes:
  - ./caddy-root.crt:/etc/ssl/certs/caddy-root.crt:ro
```

`NODE_EXTRA_CA_CERTS` tells the Node.js runtime used by n8n to trust Caddy's root certificate in addition to its normal trusted authorities.

## 4. Recreate the n8n container

First check the rendered Compose configuration:

```bash
docker compose config
```

Recreate n8n so the environment variable is applied when Node.js starts:

```bash
docker compose up -d --force-recreate n8n
```

A normal restart may not apply changes made to `compose.yaml`.

Confirm that the running container has the variable:

```bash
docker exec n8n printenv NODE_EXTRA_CA_CERTS
```

Expected result:

```text
/etc/ssl/certs/caddy-root.crt
```

Confirm that the certificate is mounted:

```bash
docker exec n8n ls -l /etc/ssl/certs/caddy-root.crt
```

If the environment-variable command returns nothing, the container was not created from the modified Compose configuration. Find the Compose file used to create it with:

```bash
docker inspect n8n \
  --format '{{ index .Config.Labels "com.docker.compose.project.config_files" }}'
```

Its working directory can be found with:

```bash
docker inspect n8n \
  --format '{{ index .Config.Labels "com.docker.compose.project.working_dir" }}'
```

## 5. Test HTTPS from Node.js

Test using Node.js rather than relying only on `wget`, because Node.js is the runtime used by n8n and reads `NODE_EXTRA_CA_CERTS`:

```bash
docker exec n8n node -e 'require("https").get("https://ollama.home/api/tags", response => {
  console.log("HTTP status:", response.statusCode);
  response.pipe(process.stdout);
}).on("error", console.error);'
```

The working result is:

```text
HTTP status: 200
{"models":[...]}
```

This confirms all of the following:

- Docker can resolve `ollama.home`.
- The n8n container can reach Caddy.
- Node.js trusts Caddy's certificate.
- Caddy can reach Ollama.
- The Ollama API is responding.

## 6. Fix the `Not Found` error

After HTTPS works, n8n may still report:

```text
Error details: Not Found
```

This occurs when the base URL is entered as:

```text
https://ollama.home
```

The n8n Assistant's self-hosted option expects an **OpenAI-compatible** endpoint. Ollama exposes that API beneath `/v1`, so the correct base URL is:

```text
https://ollama.home/v1
```

Without `/v1`, n8n requests a route such as `/models`, which Ollama does not provide. The correct OpenAI-compatible route is `/v1/models`.

Test it directly from the n8n container:

```bash
docker exec n8n node -e 'require("https").get("https://ollama.home/v1/models", response => {
  console.log("HTTP status:", response.statusCode);
  response.pipe(process.stdout);
}).on("error", console.error);'
```

## 7. Final n8n model settings

In **Connect a model**, use:

| Setting | Value |
| --- | --- |
| Provider | Self-hosted or OpenAI-compatible endpoint |
| Base URL | `https://ollama.home/v1` |
| API key | `ollama` |
| Model | An installed model name, for example `qwen3.8:latest` |

The key is a required placeholder for the OpenAI client; it is not an Ollama authentication credential.

## Troubleshooting summary

| Error | Cause | Fix |
| --- | --- | --- |
| `OpenAI API key is missing` | The OpenAI-compatible client requires a non-empty key | Enter `ollama` as a placeholder |
| `unable to get local issuer certificate` | Node.js inside n8n does not trust Caddy's CA | Mount `caddy-root.crt` and set `NODE_EXTRA_CA_CERTS` |
| `wget: Connection reset by peer` | The command-line client in the container does not trust the private CA | Validate with Node.js and install CA trust if `wget` itself must work |
| `NODE_EXTRA_CA_CERTS` is blank | The running container does not have the updated Compose environment | Edit the correct Compose file and force-recreate n8n |
| `Not Found` | The OpenAI-compatible `/v1` prefix is missing | Use `https://ollama.home/v1` |
| `HTTP status: 200` from `/api/tags` | Native Ollama API is reachable | Continue with the `/v1/models` test |

## Security note

Ollama does not provide built-in API authentication for a local server. Keep the Caddy endpoint restricted to the trusted local network, do not expose it through router port forwarding, and add an authentication layer if remote access is ever required.
