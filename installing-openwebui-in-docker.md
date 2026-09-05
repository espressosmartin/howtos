# Installing Open WebUI in Docker

Open WebUI provides a web interface for running and interacting with local AI models. One of the cleanest ways to run it is with Docker, which keeps Open WebUI and its dependencies isolated from the host operating system and makes upgrades and backups straightforward.

This guide installs Open WebUI using Docker Compose with persistent storage and connects it to an existing Ollama server.

## 1. Prerequisites

This guide assumes you already have Docker and Docker Compose installed.

Check Docker:

```bash
docker --version
```

Check Docker Compose:

```bash
docker compose version
```

You will also need an Ollama server accessible from the machine running Open WebUI.

For example:

```text
http://OLLAMA_SERVER_IP:11434
```

You can test it with:

```bash
curl http://OLLAMA_SERVER_IP:11434/api/tags
```

If Ollama is working, this should return JSON containing the installed models.

---

## 2. Create the Open WebUI Directory

Create a directory for the Docker installation:

```bash
mkdir -p ~/docker/open-webui/data
cd ~/docker/open-webui
```

The structure will eventually look like:

```text
open-webui/
├── compose.yml
└── data/
```

The `data` directory will contain Open WebUI's persistent data, while `compose.yml` defines the container.

---

## 3. Create the Docker Compose File

Create:

```bash
nano compose.yml
```

Add:

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui

    ports:
      - "8080:8080"

    volumes:
      - ./data:/app/backend/data

    environment:
      OLLAMA_BASE_URL: http://OLLAMA_SERVER_IP:11434

    restart: unless-stopped
```

Replace:

```text
OLLAMA_SERVER_IP
```

with the hostname or IP address of your Ollama server.

For example:

```text
http://ollama.home:11434
```

or:

```text
http://192.168.x.x:11434
```

---

## 4. Understanding the Configuration

### Docker image

```yaml
image: ghcr.io/open-webui/open-webui:main
```

This pulls the Open WebUI Docker image from GitHub's container registry.

The `main` tag follows the current release stream. For a more controlled installation, a specific Open WebUI version can be used instead.

### Port

```yaml
ports:
  - "8080:8080"
```

This makes Open WebUI available on port `8080` of the Docker host.

Initially, it can therefore be accessed at:

```text
http://SERVER_IP:8080
```

### Persistent storage

```yaml
volumes:
  - ./data:/app/backend/data
```

This is particularly important.

Inside the container, Open WebUI stores its persistent data in:

```text
/app/backend/data
```

The bind mount stores that data on the host in:

```text
~/docker/open-webui/data
```

Removing or replacing the Docker container therefore does not delete your Open WebUI data.

It also makes backups straightforward.

### Ollama connection

```yaml
environment:
  OLLAMA_BASE_URL: http://OLLAMA_SERVER_IP:11434
```

This tells Open WebUI where to find Ollama.

### Automatic restart

```yaml
restart: unless-stopped
```

Docker will restart Open WebUI after a reboot or unexpected failure unless you have explicitly stopped the container.

---

## 5. Start Open WebUI

Pull the image:

```bash
docker compose pull
```

Start the container:

```bash
docker compose up -d
```

The `-d` option runs it in the background.

Check its status:

```bash
docker compose ps
```

You should see the `open-webui` container running.

---

## 6. Check the Logs

Watch the Open WebUI startup:

```bash
docker compose logs -f
```

Or specifically:

```bash
docker compose logs -f open-webui
```

Press:

```text
Ctrl+C
```

to leave the log viewer.

This does **not** stop Open WebUI.

---

## 7. Open Open WebUI

Visit:

```text
http://SERVER_IP:8080
```

If you're using the browser on the Docker machine itself:

```text
http://localhost:8080
```

The first account created normally becomes the administrator account, so choose its credentials carefully.

---

## 8. Test the Ollama Connection

A useful test is to query Ollama from **inside the Open WebUI container**:

```bash
docker exec open-webui \
  curl -s http://OLLAMA_SERVER_IP:11434/api/tags
```

A successful response should contain the models installed in Ollama:

```json
{
  "models": [
    {
      "name": "qwen3:latest"
    }
  ]
}
```

This proves that:

```text
Open WebUI container
        |
        v
Docker network
        |
        v
Ollama server :11434
```

is working.

---

## 9. Configure Ollama Inside Open WebUI

If Open WebUI starts but doesn't show the Ollama models, check the Ollama connection in Open WebUI's administration settings.

Set the URL to:

```text
http://OLLAMA_SERVER_IP:11434
```

Open WebUI can persist connection settings in its database, so the value configured through the UI may matter even if `OLLAMA_BASE_URL` is present in Docker Compose.

After saving the connection, the Ollama models should become available in the model selector.

---

## 10. If Ollama Is Running on the Docker Host

There is an important Docker networking detail if Ollama and Open WebUI are running on the same computer.

Inside the Open WebUI container:

```text
localhost
```

means **the Open WebUI container**, not the Linux host.

Therefore this may not work:

```text
http://localhost:11434
```

One approach on Linux is to add:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

and configure:

```yaml
environment:
  OLLAMA_BASE_URL: http://host.docker.internal:11434
```

The complete configuration would then look like:

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui

    ports:
      - "8080:8080"

    volumes:
      - ./data:/app/backend/data

    extra_hosts:
      - "host.docker.internal:host-gateway"

    environment:
      OLLAMA_BASE_URL: http://host.docker.internal:11434

    restart: unless-stopped
```

Ollama must also be listening on an interface accessible from Docker rather than only `127.0.0.1`.

---

## 11. Useful Docker Commands

### Start Open WebUI

```bash
cd ~/docker/open-webui
docker compose up -d
```

### Stop Open WebUI

```bash
docker compose down
```

### Restart Open WebUI

```bash
docker compose restart
```

### Check status

```bash
docker compose ps
```

### View logs

```bash
docker compose logs -f
```

### View recent logs

```bash
docker compose logs --tail=100
```

### Check the container directly

```bash
docker ps
```

---

## 12. Updating Open WebUI

Before updating, back up the `data` directory.

Then:

```bash
cd ~/docker/open-webui
docker compose pull
docker compose up -d
```

Docker downloads the newer image and recreates the container if necessary.

Because the persistent data lives outside the container:

```text
./data
```

your conversations, users and configuration survive container replacement.

Check the logs afterwards:

```bash
docker compose logs -f
```

For a server where stability matters, consider pinning the Docker image to a specific Open WebUI version rather than permanently using:

```yaml
image: ghcr.io/open-webui/open-webui:main
```

This gives you more control over when upgrades happen.

---

## 13. Backing Up Open WebUI

The important persistent data is stored in:

```text
~/docker/open-webui/data
```

A simple safe backup is to stop Open WebUI:

```bash
docker compose down
```

Archive the installation:

```bash
tar -czf \
  open-webui-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
  compose.yml data
```

Then restart:

```bash
docker compose up -d
```

Keep backups somewhere other than the disk hosting Open WebUI if you want protection against disk failure.

---

## 14. Troubleshooting

### Open WebUI isn't running

Check:

```bash
docker compose ps
```

Then:

```bash
docker compose logs --tail=100
```

### Check port 8080

```bash
ss -ltnp | grep 8080
```

You can also test it locally:

```bash
curl -I http://localhost:8080
```

### Ollama models don't appear

First test Ollama from the Docker host:

```bash
curl http://OLLAMA_SERVER_IP:11434/api/tags
```

Then test from the container:

```bash
docker exec open-webui \
  curl -s http://OLLAMA_SERVER_IP:11434/api/tags
```

If the first succeeds but the second fails, investigate Docker networking or firewall rules.

If both succeed, check Open WebUI's Ollama connection configuration.

### Connection refused from Ollama

On the Ollama server:

```bash
ss -ltnp | grep 11434
```

If Ollama is only listening on:

```text
127.0.0.1:11434
```

it isn't accessible remotely.

For a systemd installation, this can be changed with:

```bash
sudo systemctl edit ollama
```

and:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

If Ollama is exposed to the network, use firewall rules to restrict access to trusted machines.

### Permission problems with the data directory

Check:

```bash
ls -lah ~/docker/open-webui/data
```

and:

```bash
docker compose logs --tail=100
```

Avoid blindly making the directory world-writable. Determine which UID/GID the container expects and correct the ownership if necessary.

---

## 15. Adding HTTPS

At this stage the architecture is:

```text
Browser
   |
   | HTTP :8080
   v
Open WebUI
   |
   | HTTP :11434
   v
 Ollama
```

For basic use on a trusted local network this works, but some browser functionality — particularly microphone and media-device APIs — requires a secure HTTPS context.

A reverse proxy such as **Caddy** can provide this:

```text
Browser
   |
   | HTTPS :443
   v
 Caddy
   |
   | HTTP :8080
   v
Open WebUI
   |
   | HTTP :11434
   v
 Ollama
```

For example, with a local hostname:

```text
https://openwebui.home
```

Caddy can terminate HTTPS and reverse-proxy requests to Open WebUI.

That can be treated as a separate step from the basic Docker installation.

---

# Final Configuration

The basic Docker installation consists of:

```text
open-webui/
├── compose.yml
└── data/
```

with:

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui

    ports:
      - "8080:8080"

    volumes:
      - ./data:/app/backend/data

    environment:
      OLLAMA_BASE_URL: http://OLLAMA_SERVER_IP:11434

    restart: unless-stopped
```

This provides a simple separation between the application and its data:

```text
Docker image
     |
     | disposable
     v
Open WebUI container
     |
     | persistent
     v
./data
```

The container can be stopped, upgraded or recreated without losing Open WebUI's persistent data.

From here, Ollama provides the models, while a reverse proxy such as Caddy can be added to provide friendly local hostnames and trusted HTTPS.
