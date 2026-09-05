# Migrating Open WebUI from pip to Docker and Backing It Up

If Open WebUI was originally installed using `pip` or inside a Conda environment, moving it to Docker makes ongoing maintenance considerably easier.

The important thing to understand is that **Open WebUI's application and its data are separate**. You don't need to migrate the Python installation itself. You need to preserve Open WebUI's data directory, which contains the database, uploads and other persistent state.

This guide covers:

* Finding an existing pip/Conda Open WebUI installation
* Backing it up safely
* Migrating the data to Docker
* Verifying the migration
* Removing the old Python installation
* Creating ongoing backups of the Docker installation
* Restoring from a backup

---

# 1. Find the Existing Open WebUI Data

First find the Open WebUI database:

```bash id="q6qlxy"
find ~ -name webui.db 2>/dev/null
```

If Open WebUI is installed in Conda, you can narrow the search:

```bash id="48szkr"
find ~/miniconda3 ~/anaconda3 \
  -name webui.db 2>/dev/null
```

A pip/Conda installation might return something similar to:

```text id="3b32pp"
~/miniconda3/envs/open-webui/lib/python3.11/site-packages/open_webui/data/webui.db
```

The important part is the directory containing `webui.db`:

```text id="yjm9mq"
.../open_webui/data/
```

Do **not** migrate only `webui.db`.

The data directory can also contain uploads, vector databases, cache data and other persistent Open WebUI state.

Inspect it:

```bash id="6h1idn"
ls -lah /PATH/TO/open_webui/data/
```

---

# 2. Stop Open WebUI

Before copying the database, stop the pip version of Open WebUI.

If it is running in the current terminal:

```text id="zqvv1s"
Ctrl+C
```

Check that it has stopped:

```bash id="q6azsv"
pgrep -af open-webui
```

You can also check whether anything is still listening on Open WebUI's usual port:

```bash id="tixs88"
ss -ltnp | grep 8080
```

It's best to back up the database while Open WebUI isn't running so that SQLite isn't being modified during the copy.

---

# 3. Make a Backup Before Migrating

Create a backup directory:

```bash id="g60g1k"
mkdir -p ~/backups/open-webui
```

Then copy the entire Open WebUI data directory:

```bash id="l75qnf"
cp -a \
  /PATH/TO/open_webui/data \
  ~/backups/open-webui/data-$(date +%Y%m%d-%H%M%S)
```

For example, the resulting directory might be:

```text id="s12pfr"
~/backups/open-webui/data-20260905-120000/
```

Check it:

```bash id="iwxg4r"
ls -lah ~/backups/open-webui/
```

It is worth checking that the database exists inside the backup:

```bash id="o4nhpw"
find ~/backups/open-webui -name webui.db -ls
```

At this point you have a rollback copy of the original pip installation's data.

---

# 4. Create the Docker Installation

Create a directory for Open WebUI:

```bash id="v6pvyu"
mkdir -p ~/docker/open-webui/data
cd ~/docker/open-webui
```

Create `compose.yml`:

```yaml id="p9w9mh"
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui

    ports:
      - "8080:8080"

    volumes:
      - ./data:/app/backend/data

    environment:
      OLLAMA_BASE_URL: http://OLLAMA_SERVER:11434

    restart: unless-stopped
```

The important part for migration and backups is:

```yaml id="wqpd08"
volumes:
  - ./data:/app/backend/data
```

This maps Open WebUI's persistent data onto:

```text id="q8y6qd"
~/docker/open-webui/data
```

on the host.

That makes the data easy to inspect, migrate and back up.

---

# 5. Copy the Existing Data

With both the old pip Open WebUI and new Docker Open WebUI stopped, copy the contents of the old data directory:

```bash id="xcn8ya"
cp -a \
  /PATH/TO/open_webui/data/. \
  ~/docker/open-webui/data/
```

The `/.` is deliberate. It copies the **contents** of `data` into the new `data` directory rather than creating:

```text id="b8yxmn"
~/docker/open-webui/data/data/
```

Check the result:

```bash id="lfupja"
ls -lah ~/docker/open-webui/data/
```

You should see:

```text id="oc3e1n"
webui.db
```

along with any other persistent directories/files from the old installation.

---

# 6. Start the Docker Version

Start Open WebUI:

```bash id="onbxb3"
cd ~/docker/open-webui

docker compose pull
docker compose up -d
```

Watch the initial startup:

```bash id="g84j5u"
docker compose logs -f
```

Open WebUI may perform database migrations when starting a newer version against an older database.

Once it has started successfully, open Open WebUI in your browser.

Check that important data has survived, particularly:

* Your user account
* Previous conversations
* Settings
* Knowledge/documents
* Model configuration
* Connections
* Uploaded files

---

# 7. Check External Connections

One thing that may need changing after migration is the Ollama URL.

A pip installation might have been able to use:

```text id="knv4rb"
http://localhost:11434
```

Inside Docker, `localhost` refers to the **Open WebUI container itself**, not the Docker host or another machine.

If Ollama is on another machine, configure:

```text id="b83uz1"
http://OLLAMA_SERVER:11434
```

You can test connectivity directly from inside Open WebUI:

```bash id="52zg59"
docker exec open-webui \
  curl -s http://OLLAMA_SERVER:11434/api/tags
```

If you receive JSON containing the Ollama models, connectivity is working.

Also check Open WebUI's connection configuration in its administration settings.

Open WebUI can persist connection information in `webui.db`, so settings migrated from the old installation may take precedence over defaults supplied through Docker environment variables.

---

# 8. Don't Delete the Old Installation Yet

Keep the pip/Conda installation and original backup until the Docker version has been running successfully for a while.

Once you're satisfied with the migration, identify the old installation:

```bash id="fww82f"
which open-webui
```

and:

```bash id="vpjv9m"
python -m pip show open-webui
```

If Open WebUI was installed into a dedicated Conda environment:

```bash id="4kysag"
conda env list
```

Once you're certain it is no longer needed, the dedicated environment can be removed:

```bash id="3nphng"
conda remove -n open-webui --all
```

If it was installed directly with pip:

```bash id="6thahz"
pip uninstall open-webui
```

Keep at least one known-good backup even after removing the old installation.

---

# 9. Backing Up Docker Open WebUI

With this Docker configuration, persistent Open WebUI data lives in:

```text id="jynv9f"
~/docker/open-webui/data/
```

That makes backups straightforward.

## Safest method: stop, archive, start

Stop Open WebUI:

```bash id="qz69b2"
cd ~/docker/open-webui
docker compose down
```

Create a compressed backup:

```bash id="e5eylf"
tar -czf \
  ~/backups/open-webui/open-webui-$(date +%Y%m%d-%H%M%S).tar.gz \
  -C ~/docker/open-webui \
  data
```

Start Open WebUI again:

```bash id="ejn5y2"
docker compose up -d
```

You now have something like:

```text id="nzzp95"
open-webui-20260905-120000.tar.gz
```

containing the entire persistent data directory.

---

# 10. Back Up the Docker Configuration Too

The persistent data isn't the only thing worth preserving.

Also back up:

```text id="ic0mm1"
compose.yml
```

and any associated:

```text id="kt5q42"
.env
```

file.

For example, archive the whole installation directory while Open WebUI is stopped:

```bash id="g0f7ai"
tar -czf \
  ~/backups/open-webui/open-webui-full-$(date +%Y%m%d-%H%M%S).tar.gz \
  -C ~/docker \
  open-webui
```

This has the advantage of preserving:

```text id="3qgdzr"
open-webui/
├── compose.yml
├── .env
└── data/
    ├── webui.db
    └── ...
```

Be aware that `.env` files can contain passwords, API keys or other secrets. Backups containing them should be protected accordingly.

---

# 11. Verify Your Backups

A backup that has never been checked isn't something to rely on.

List the contents:

```bash id="12cp28"
tar -tzf ~/backups/open-webui/open-webui-full-*.tar.gz | head -50
```

Make sure you can see:

```text id="f3hx0p"
open-webui/compose.yml
open-webui/data/
open-webui/data/webui.db
```

You can also test the archive:

```bash id="kgxg95"
gzip -t ~/backups/open-webui/open-webui-full-YYYYMMDD-HHMMSS.tar.gz
```

No output generally means the gzip archive passed its integrity check.

---

# 12. Restoring Open WebUI

Suppose the existing installation has been lost and you have:

```text id="mrh4ze"
open-webui-full-YYYYMMDD-HHMMSS.tar.gz
```

Stop Open WebUI if it exists:

```bash id="r3y8un"
cd ~/docker/open-webui
docker compose down
```

Move the broken installation out of the way rather than immediately deleting it:

```bash id="9pj0gy"
mv ~/docker/open-webui \
   ~/docker/open-webui-old
```

Extract the backup:

```bash id="nw6xgb"
tar -xzf \
  ~/backups/open-webui/open-webui-full-YYYYMMDD-HHMMSS.tar.gz \
  -C ~/docker
```

Check:

```bash id="tz0suw"
ls -lah ~/docker/open-webui/data/
```

Then:

```bash id="2fv0z8"
cd ~/docker/open-webui
docker compose up -d
```

Watch the logs:

```bash id="2sy5mq"
docker compose logs -f
```

---

# 13. A Simple Backup Script

For a personal server, a small script can automate the process.

Create:

```text id="6mb53c"
backup-open-webui.sh
```

with:

```bash id="zhuzd4"
#!/usr/bin/env bash

set -e

OPENWEBUI_DIR="$HOME/docker/open-webui"
BACKUP_DIR="$HOME/backups/open-webui"
TIMESTAMP="$(date +%Y%m%d-%H%M%S)"

mkdir -p "$BACKUP_DIR"

cd "$OPENWEBUI_DIR"

echo "Stopping Open WebUI..."
docker compose down

echo "Creating backup..."
tar -czf \
  "$BACKUP_DIR/open-webui-$TIMESTAMP.tar.gz" \
  -C "$(dirname "$OPENWEBUI_DIR")" \
  "$(basename "$OPENWEBUI_DIR")"

echo "Starting Open WebUI..."
docker compose up -d

echo "Backup created:"
echo "$BACKUP_DIR/open-webui-$TIMESTAMP.tar.gz"
```

Make it executable:

```bash id="v1c7yk"
chmod +x backup-open-webui.sh
```

Run:

```bash id="j4xjrx"
./backup-open-webui.sh
```

---

# 14. Don't Let Backups Accumulate Forever

If backups are created regularly, old archives can consume significant disk space.

For example, delete backups older than 30 days:

```bash id="8b2a33"
find ~/backups/open-webui \
  -type f \
  -name 'open-webui-*.tar.gz' \
  -mtime +30 \
  -delete
```

A sensible strategy might be:

```text id="mtmtus"
Daily backups       → keep 7
Weekly backups      → keep 4
Monthly backups     → keep 6–12
```

More importantly, keep at least one copy **off the machine running Open WebUI**.

A backup on the same SSD protects against accidental deletion or a bad upgrade, but it doesn't protect against disk failure.

---

# 15. Back Up Before Updating Open WebUI

Before:

```bash id="x8xq66"
docker compose pull
docker compose up -d
```

create a backup.

This is particularly important because a newer Open WebUI release may migrate the database when it starts.

A good update procedure is:

```bash id="i4pz9k"
cd ~/docker/open-webui

docker compose down

tar -czf \
  ~/backups/open-webui/pre-update-$(date +%Y%m%d-%H%M%S).tar.gz \
  -C ~/docker \
  open-webui

docker compose pull
docker compose up -d

docker compose logs -f
```

If something goes wrong, you have a snapshot of the installation from immediately before the upgrade.

---

# What Actually Needs Backing Up?

With a bind-mounted Docker installation:

```yaml id="m4h4tx"
volumes:
  - ./data:/app/backend/data
```

the most important directory is:

```text id="mh1y0s"
data/
```

In particular:

```text id="yr1m6k"
data/webui.db
```

contains a large amount of Open WebUI's persistent state.

But don't make the mistake of backing up only `webui.db`.

Back up the **entire data directory**, because Open WebUI can also store uploaded files and other persistent resources alongside the database.

Ideally, preserve the whole installation:

```text id="agqjy9"
open-webui/
├── compose.yml
├── .env
└── data/
```

That gives you everything needed to rebuild the container quickly.

The Docker image itself does **not** need backing up. It can simply be pulled again from the container registry.

---

# Recommended Backup Strategy

For a home Open WebUI server, a practical strategy is:

1. Keep Open WebUI's `/app/backend/data` on a host bind mount.
2. Back up the entire bind-mounted directory rather than only `webui.db`.
3. Back up `compose.yml` and `.env` as well.
4. Stop Open WebUI briefly while taking a full filesystem backup.
5. Always create a backup immediately before upgrading.
6. Keep multiple generations rather than overwriting the previous backup.
7. Keep at least one copy on another physical machine or storage device.
8. Occasionally perform a test restore.

With that in place, recovering Open WebUI is essentially:

```text id="7b93ng"
Install Docker
      ↓
Restore open-webui/
      ↓
docker compose up -d
      ↓
Open WebUI restored
```

That simplicity is one of the main advantages of moving a pip-based Open WebUI installation into Docker.
