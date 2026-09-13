# Monitoring a Home Network with FRITZ!Box, Deco and ntopng

This guide configures ntopng on an Ubuntu computer to analyse internet traffic captured by a FRITZ!Box. It is intended for a network where:

- the FRITZ!Box is the router, firewall and DHCP server;
- TP-Link Deco runs in **Access Point mode**;
- client addresses are in the `192.168.178.0/24` range;
- ntopng runs on a Linux computer called `fractal` at `192.168.178.108`.

The FRITZ!Box is the network gateway, so internet traffic from Deco clients passes through it. FRITZ!OS includes a diagnostic packet-capture facility. The ntop project supplies `fritzdump.sh`, which authenticates to the router, starts that capture and pipes its PCAP stream into ntopng.

> This integration script is maintained by ntop, not by FRITZ/AVM. It is best treated as an experiment or diagnostic setup before relying on it continuously.

## What this can monitor

When the LAN-side capture works, ntopng can show:

- individual `192.168.178.x` clients;
- upload and download volumes;
- active and historical flows;
- remote destinations;
- detected application protocols;
- top talkers and network alerts.

It may not see traffic that remains entirely inside a Deco access point or elsewhere in the local mesh without passing through the FRITZ!Box.

## Prerequisites

- ntopng installed and working on Ubuntu;
- Redis installed/running as required by ntopng;
- access to the FRITZ!Box administration interface;
- Deco confirmed as **Access Point** under **More → Advanced → Operation Mode**;
- the Linux host and clients on the same `192.168.178.x` network.

Check the normal ntopng service:

```bash
sudo systemctl status ntopng --no-pager
```

The packaged service in this setup normally launches as:

```text
/usr/bin/ntopng /run/ntopng.conf
```

and provides its normal interface at:

```text
https://192.168.178.108:3001/
```

## 1. Create a dedicated FRITZ!Box user

In the FRITZ!Box interface, open:

**System → FRITZ!Box Users → Add User**

Create a separate account for capture access:

- use a unique username and strong password;
- allow the permissions required to view/change FRITZ!Box settings;
- do not grant that user internet access to the FRITZ!Box.

Using a dedicated account avoids exposing the primary administrator password to the capture script.

## 2. Download the official ntop Fritz capture script

```bash
sudo install -d -m 700 /opt/ntopng-fritz

sudo curl -fsSL \
  https://raw.githubusercontent.com/ntop/ntopng/dev/tools/fritzdump.sh \
  -o /opt/ntopng-fritz/fritzdump.sh

sudo chmod 700 /opt/ntopng-fritz/fritzdump.sh
```

## 3. Select the LAN-side FRITZ!Box capture

The downloaded script defaults to the WAN interface:

```bash
IFACE="2-0"
```

For per-device monitoring, change it to the LAN side so private client addresses remain visible:

```bash
sudo sed -i \
  's/^IFACE="2-0"/IFACE="1-lan"/' \
  /opt/ntopng-fritz/fritzdump.sh
```

Confirm it:

```bash
grep '^IFACE=' /opt/ntopng-fritz/fritzdump.sh
```

Expected:

```text
IFACE="1-lan"
```

## 4. Copy the working ntopng runtime configuration

Do this while the normal ntopng service is still running. Its systemd unit deletes `/run/ntopng.conf` when stopped.

```bash
sudo cp /run/ntopng.conf /etc/ntopng/ntopng-fritz.conf
sudo chmod 600 /etc/ntopng/ntopng-fritz.conf
```

Remove the original physical-interface entry from the copy. The Fritz PCAP stream will replace it:

```bash
sudo sed -i -E \
  '/^[[:space:]]*(-i|--interface)(=|[[:space:]])/d' \
  /etc/ntopng/ntopng-fritz.conf
```

## 5. Make `fritzdump.sh` use the working ntopng configuration

A bare `ntopng -i -` launch did not locate the packaged web assets correctly and produced a recursive login redirect. The known-working command supplies the runtime configuration and explicitly identifies the installed web directories.

Update the final command in the script:

```bash
sudo sed -i \
  's#| /usr/bin/ntopng.*$#| /usr/bin/ntopng /etc/ntopng/ntopng-fritz.conf --install-dir /usr/share/ntopng --httpdocs-dir /usr/share/ntopng/httpdocs --scripts-dir /usr/share/ntopng/scripts -i -#' \
  /opt/ntopng-fritz/fritzdump.sh
```

If the original line still ends in the shorter `ntopng -i -`, use this more general replacement instead:

```bash
sudo sed -i \
  's#| ntopng -i -$#| /usr/bin/ntopng /etc/ntopng/ntopng-fritz.conf --install-dir /usr/share/ntopng --httpdocs-dir /usr/share/ntopng/httpdocs --scripts-dir /usr/share/ntopng/scripts -i -#' \
  /opt/ntopng-fritz/fritzdump.sh
```

Verify the result:

```bash
tail -n 1 /opt/ntopng-fritz/fritzdump.sh
```

It should end with:

```bash
| /usr/bin/ntopng /etc/ntopng/ntopng-fritz.conf --install-dir /usr/share/ntopng --httpdocs-dir /usr/share/ntopng/httpdocs --scripts-dir /usr/share/ntopng/scripts -i -
```

## 6. Stop the standard ntopng instance

Only one ntopng instance can use the web ports at a time:

```bash
sudo systemctl stop ntopng
pgrep -a ntopng
```

No ntopng process should remain.

## 7. Start the FRITZ!Box capture

Prompt for credentials so the password is not written into shell history:

```bash
read -rp "Fritz username: " FRITZ_USER
read -rsp "Fritz password: " FRITZ_PASSWORD
echo

sudo /opt/ntopng-fritz/fritzdump.sh \
  "$FRITZ_USER" "$FRITZ_PASSWORD"

unset FRITZ_USER FRITZ_PASSWORD
```

Keep this terminal open. The script and ntopng run in the foreground.

The upstream script still passes the password as a process argument briefly. Use the dedicated FRITZ!Box account rather than the main administrator account.

## 8. Open ntopng

Check which web port is active:

```bash
sudo ss -lntp | grep -E ':(3000|3001)\b'
pgrep -a ntopng
```

In the successfully tested interactive setup, the working page was:

```text
http://192.168.178.108:3000/lua/index.lua
```

If the configuration instead starts the packaged HTTPS listener, use:

```text
https://192.168.178.108:3001/
```

Accept the local/self-signed certificate warning if HTTPS is used.

Do not expose either ntopng address to the public internet. HTTP port `3000` is suitable only for a temporary test on a trusted local network because login details and traffic are not encrypted.

## 9. Confirm that whole-network capture is working

Generate obvious traffic from a phone or computer connected to Deco, such as starting a download or video stream. In ntopng, check:

**Hosts → Hosts** and **Flows → Live Flows**

The test succeeds when ntopng shows several individual `192.168.178.x` clients communicating with remote internet addresses. If it mainly shows `fractal`, multicast and broadcast traffic, it is still capturing the Linux host's physical interface rather than the FRITZ!Box stream.

Remember that a hostname such as `openwebui.local` describes the whole Linux host at that IP address; it does not prove that every flow came from Open WebUI.

## Troubleshooting

### Too many redirects

The observed failure repeatedly nested these paths:

```text
/lua/login.lua
/lua/http_status_code.lua?message=not_found
```

The installed Lua files did exist:

```text
/usr/share/ntopng/scripts/lua/login.lua
/usr/share/ntopng/scripts/lua/http_status_code.lua
```

The explicit `--install-dir`, `--httpdocs-dir` and `--scripts-dir` arguments in the known-working command corrected the runtime lookup problem.

Check the redirect chain with:

```bash
curl -sS -L --max-redirs 5 \
  -o /dev/null \
  -w 'Status: %{http_code}; redirects: %{num_redirects}; final: %{url_effective}\n' \
  http://127.0.0.1:3000/
```

For HTTPS:

```bash
curl -k -sS -L --max-redirs 5 \
  -o /dev/null \
  -w 'Status: %{http_code}; redirects: %{num_redirects}; final: %{url_effective}\n' \
  https://127.0.0.1:3001/
```

### Nothing is listening

```bash
pgrep -a ntopng
sudo ss -lntp | grep -E ':(3000|3001)\b'
```

If neither command shows ntopng, read the error in the terminal running `fritzdump.sh`. Likely causes include failed FRITZ!Box authentication, an invalid capture interface name, or an invalid ntopng command.

### Check that the Lua files exist

```bash
find /usr/share/ntopng -type f \
  \( -name 'login.lua' -o -name 'http_status_code.lua' \) \
  -print
```

### Check the normal packaged service

Stop the interactive capture with `Ctrl+C`, then:

```bash
sudo systemctl restart ntopng
pgrep -a ntopng
sudo ss -lntp | grep -E ':(3000|3001)\b'
```

The normal service should again be available at its original address, normally:

```text
https://192.168.178.108:3001/
```

### Restore the known-working script command

If later experiments alter the script, restore it with:

```bash
sudo sed -i \
  's#| /usr/bin/ntopng.*$#| /usr/bin/ntopng /etc/ntopng/ntopng-fritz.conf --install-dir /usr/share/ntopng --httpdocs-dir /usr/share/ntopng/httpdocs --scripts-dir /usr/share/ntopng/scripts -i -#' \
  /opt/ntopng-fritz/fritzdump.sh
```

## Stop the experiment and return to normal ntopng

In the capture terminal, press:

```text
Ctrl+C
```

Then restore the packaged service:

```bash
sudo systemctl start ntopng
sudo systemctl status ntopng --no-pager
```

## Security and operational cautions

- Packet captures contain sensitive network metadata and may contain the contents of unencrypted protocols.
- Keep the capture and ntopng interface restricted to the trusted home LAN.
- Do not add FRITZ!Box port sharing for ntopng.
- Prefer WireGuard when remote access is required.
- Use a dedicated, non-remote FRITZ!Box account for the capture script.
- Continuous capture can add load to the FRITZ!Box and duplicate traffic onto the LAN.
- FRITZ!OS or ntopng updates may change the undocumented details used by the live script.
- For permanent monitoring, a managed switch with port mirroring and a dedicated capture adapter on `fractal` is a cleaner design.

## References

- ntop FRITZ!Box notes: <https://github.com/ntop/ntopng/blob/dev/doc/README.fritzbox>
- ntop `fritzdump.sh`: <https://github.com/ntop/ntopng/blob/dev/tools/fritzdump.sh>
- ntopng documentation: <https://www.ntop.org/guides/ntopng/>
- TP-Link Deco Access Point mode: <https://www.tp-link.com/us/support/faq/1842/>
