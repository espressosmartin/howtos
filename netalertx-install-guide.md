# NetAlertX installation on Ubuntu machine

This guide records the working Docker installation of NetAlertX on the Ubuntu machine.

## Purpose

NetAlertX provides:

- discovery of devices on the home network;
- a persistent inventory of IP and MAC addresses;
- first-seen and last-seen information;
- alerts when new devices appear;
- presence monitoring for important devices.

NetAlertX is **not** a traffic or bandwidth monitor. Continue using ntopng for traffic flows and Pi-hole for DNS-query history.

## Network details

| Setting | Value |
| --- | --- |
| NetAlertX host | `Ubuntu` |
| Host IP address | `192.168.178.108` |
| Home subnet | `192.168.178.0/24` |
| Physical interface | `enp113s0` |
| Web interface port | `20211` |
| GraphQL/API port | `20212` |
| Data directory | `/home/martin/docker/netalertx/data` |

The container uses Docker host networking. This is required so ARP scanning can reach devices on the physical LAN, including clients connected through the FRITZ!Box and Deco access points.

## 1. Create the directories

```bash
mkdir -p /home/martin/docker/netalertx/data
cd /home/martin/docker/netalertx
```

Give NetAlertX's default container user ownership of the data directory:

```bash
sudo chown -R 20211:20211 /home/martin/docker/netalertx/data
```

## 2. Create `docker-compose.yml`

Create `/home/martin/docker/netalertx/docker-compose.yml` with the following content:

```yaml
services:
  netalertx:
    container_name: netalertx
    image: ghcr.io/netalertx/netalertx:latest

    # Required for ARP discovery on the physical LAN
    network_mode: host

    cap_add:
      - NET_RAW
      - NET_ADMIN
      - NET_BIND_SERVICE

    environment:
      PORT: "20211"
      LOADED_PLUGINS: "[\"ARPSCAN\"]"
      APP_CONF_OVERRIDE: "{\"GRAPHQL_PORT\":\"20212\",\"SCAN_SUBNETS\":\"['192.168.178.0/24 --interface=enp113s0']\",\"ARPSCAN_RUN\":\"schedule\",\"ARPSCAN_RUN_SCHD\":\"*/5 * * * *\"}"

    volumes:
      - /home/martin/docker/netalertx/data:/data
      - /etc/localtime:/etc/localtime:ro

    tmpfs:
      - /tmp:uid=20211,gid=20211,mode=1700

    restart: unless-stopped

    logging:
      options:
        max-size: "10m"
        max-file: "3"
```

This configuration scans `192.168.178.0/24` using `enp113s0` every five minutes.

## 3. Validate and start

Always validate the YAML before starting:

```bash
cd /home/martin/docker/netalertx
docker compose config
```

Pull the image and start the container:

```bash
docker compose pull
docker compose up -d
```

Check its status:

```bash
docker compose ps
```

Open the interface from another device on the home network:

```text
http://192.168.178.108:20211
```

## 4. Restrict the web interface with UFW

If UFW is enabled, permit access from the home subnet only:

```bash
sudo ufw allow from 192.168.178.0/24 \
  to any port 20211 proto tcp \
  comment 'NetAlertX web interface'
```

Do not create a FRITZ!Box port-sharing rule for NetAlertX. Its interface should not be exposed directly to the internet.

## 5. Confirm the configuration

Display the environment override passed to the container:

```bash
docker exec netalertx printenv APP_CONF_OVERRIDE
```

Expected content:

```text
{"GRAPHQL_PORT":"20212","SCAN_SUBNETS":"['192.168.178.0/24 --interface=enp113s0']","ARPSCAN_RUN":"schedule","ARPSCAN_RUN_SCHD":"*/5 * * * *"}
```

Confirm which ports are listening:

```bash
sudo ss -lntp | grep -E ':(20211|20212)\b'
```

## 6. Test device discovery manually

Run `arp-scan` directly inside the container:

```bash
docker exec -u 0 -it netalertx \
  arp-scan --interface=enp113s0 192.168.178.0/24
```

If this lists the FRITZ!Box, Deco units, computers, phones and other devices, Docker networking and permissions are working.

The Deco system is operating as access points on the same subnet, so ordinary Deco and FRITZ!Box clients should be visible. An isolated guest network may not be visible.

## 7. First scan and device review

The interface may initially say:

```text
No devices found yet
Waiting for the first scan
Imminent
```

Wait several minutes for the first scheduled scan. When it completes, every discovered device will initially be classified as **New**. This does not mean every device is suspicious; it means NetAlertX has not seen them before.

For each new device:

1. Compare its IP and MAC address with the FRITZ!Box and Deco client lists.
2. Give it a recognisable name.
3. Mark it as known or reviewed.
4. Mark important always-on devices as favourites if desired.
5. Leave genuinely unidentified devices as new until investigated.

After the initial inventory is reviewed, future entries under **New devices** become meaningful alerts.

## 8. View logs

Show recent container output:

```bash
docker logs --tail 200 netalertx
```

Follow the output live:

```bash
docker logs --tail 200 -f netalertx
```

Press `Ctrl+C` to stop following the log without stopping NetAlertX.

List NetAlertX's internal log files:

```bash
docker exec netalertx find /tmp/log -maxdepth 2 -type f -print
```

Show the last 100 lines of each internal log:

```bash
docker exec netalertx sh -c \
  'for file in /tmp/log/*; do echo "===== $file ====="; tail -n 100 "$file"; done'
```

## 9. Troubleshooting

### Compose says `APP_CONF_OVERRIDE` must be a string

This means YAML interpreted the JSON as an object. Keep the escaped, double-quoted `APP_CONF_OVERRIDE` line exactly as shown in the Compose file and run:

```bash
docker compose config
```

Do not recreate the container until validation succeeds.

### Dashboard remains empty

First run the manual ARP scan:

```bash
docker exec -u 0 -it netalertx \
  arp-scan --interface=enp113s0 192.168.178.0/24
```

- If devices appear, host networking works; inspect the ARPSCAN plugin configuration and scheduler.
- If nothing appears, verify the interface name with `ip -o link show` and confirm that `ubuntu` can reach the LAN.

In the NetAlertX settings, verify:

- `ARPSCAN` is loaded;
- `SCAN_SUBNETS` contains `192.168.178.0/24 --interface=enp113s0`;
- `ARPSCAN_RUN` is `schedule`;
- the ARPSCAN schedule and timeout are valid.

### `Unauthorized access attempt` / API token warning

This normally means the NetAlertX browser frontend failed to authenticate with its own API. It is not normally evidence of an external attack.

After recreating the container:

1. Close all existing NetAlertX browser tabs.
2. Try `http://192.168.178.108:20211` in a private/incognito window.
3. Clear saved site data for `192.168.178.108` if the private window works.
4. Confirm that ports `20211` and `20212` are listening.

Search the logs for relevant messages:

```bash
docker logs netalertx 2>&1 | \
  grep -Ei 'graphql|api|unauthorized|20211|20212|error'
```

Inspect the saved port and confirm that an API token exists without displaying the token:

```bash
docker exec netalertx sh -c \
  "grep -E 'GRAPHQL_PORT|API_TOKEN' /data/config/app.conf | \
   sed -E 's/(API_TOKEN[^=]*=).*/\1 <hidden>/'"
```

Never post or share the actual `API_TOKEN`.

### Check for a port conflict

```bash
sudo ss -lntp | grep -E ':(20211|20212)\b'
```

The web interface should use `20211` and the NetAlertX API should use `20212`.

## 10. Common management commands

Start:

```bash
docker compose up -d
```

Stop without deleting data:

```bash
docker compose down
```

Restart:

```bash
docker compose restart
```

Recreate after changing the Compose file:

```bash
docker compose up -d --force-recreate
```

## 11. Update NetAlertX

```bash
cd /home/martin/docker/netalertx
docker compose pull
docker compose up -d --force-recreate
docker image prune
```

Check the logs after updating:

```bash
docker logs --tail 100 netalertx
```

## 12. Back up the configuration and device database

The persistent data is stored in:

```text
/home/martin/docker/netalertx/data
```

For a consistent backup, stop the container briefly and copy the directory:

```bash
cd /home/martin/docker/netalertx
docker compose stop

sudo rsync -aHAX --info=progress2 \
  /home/martin/docker/netalertx/data/ \
  /path/to/backup/netalertx-data/

docker compose start
```

Replace `/path/to/backup/` with the real backup destination.

## 13. Remove NetAlertX

Stop and remove the container:

```bash
cd /home/martin/docker/netalertx
docker compose down
```

The configuration and database remain in the `data` directory. Only remove that directory if the history is no longer required.

