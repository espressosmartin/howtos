# Debugging Caddy and Pi-hole on macOS

This guide covers local hostnames such as:

- `netalertx.home`
- `openwebui.home`
- `n8n.home`

Example network:

| Component | Address |
|---|---|
| Pi-hole DNS | `192.168.178.26` |
| Caddy server | `192.168.178.108` |
| NetAlertX upstream | `127.0.0.1:20211` |
| Example hostname | `netalertx.home` |

The expected request path is:

```text
Mac → Pi-hole DNS → 192.168.178.108 → Caddy → NetAlertX
```

## 1. Configure Pi-hole DNS

Add a local DNS record in Pi-hole:

```text
netalertx.home → 192.168.178.108
```

Test Pi-hole directly from the Mac:

```bash
dig +short netalertx.home @192.168.178.26
```

Expected response:

```text
192.168.178.108
```

If this fails, check the local DNS record in Pi-hole.

## 2. Test the Mac's default DNS

```bash
dig +short netalertx.home
```

Expected response:

```text
192.168.178.108
```

If querying Pi-hole directly works but this command does not, the Mac is probably using another DNS server.

Check the configured resolvers:

```bash
scutil --dns
```

Also inspect the DNS servers configured for the active network connection:

```bash
networksetup -getdnsservers Wi-Fi
```

## 3. Test the macOS system resolver

Applications such as `ping`, Safari and Chrome use the macOS system resolver. This can behave differently from `dig`.

Run:

```bash
dscacheutil -q host -a name netalertx.home
ping netalertx.home
```

A useful additional test is:

```bash
ping netalertx.home.
```

The trailing dot makes the hostname fully qualified.

If `dig` resolves correctly but `ping` reports:

```text
ping: cannot resolve netalertx.home: Unknown host
```

then Pi-hole is probably working, but macOS is not routing `.home` queries through it correctly.

## 4. Force `.home` queries through Pi-hole

Create a macOS domain-specific resolver:

```bash
sudo mkdir -p /etc/resolver
printf 'nameserver 192.168.178.26\n' | sudo tee /etc/resolver/home
```

This tells macOS to send all names ending in `.home` to Pi-hole.

The same resolver covers hostnames including:

```text
netalertx.home
openwebui.home
n8n.home
```

Flush the macOS DNS caches:

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

Test again:

```bash
dscacheutil -q host -a name netalertx.home
ping netalertx.home
```

Inspect the resolver file if needed:

```bash
cat /etc/resolver/home
```

Expected contents:

```text
nameserver 192.168.178.26
```

## 5. Configure Caddy

Example Caddy configuration:

```caddyfile
netalertx.home {
    tls internal
    reverse_proxy 127.0.0.1:20211
}
```

Validate the configuration on the Caddy server:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

Reload Caddy:

```bash
sudo systemctl reload caddy
```

Check its status:

```bash
sudo systemctl status caddy --no-pager
```

View recent Caddy logs:

```bash
sudo journalctl -u caddy -n 50 --no-pager
```

## 6. Test the application without Caddy

Run this on the Caddy/NetAlertX server:

```bash
curl -I http://127.0.0.1:20211/
```

NetAlertX may respond with a redirect such as:

```text
Location: /devices.php
```

That is normal and proves the upstream application is responding.

Check the listening ports:

```bash
sudo ss -lntp | grep -E ':(80|443|20211)\b'
```

You should normally see:

- Caddy listening on port `80` and/or `443`
- NetAlertX listening on port `20211`

If the upstream test fails, troubleshoot NetAlertX or Docker before Caddy.

## 7. Test Caddy while bypassing DNS

From the Mac:

```bash
curl -vkI \
  --resolve netalertx.home:443:192.168.178.108 \
  https://netalertx.home/
```

This forces `netalertx.home` to use `192.168.178.108`, bypassing normal DNS resolution.

If this works, then:

- Caddy is reachable
- The Caddy hostname matches
- The reverse proxy is working
- The remaining problem is DNS or the macOS resolver

A working NetAlertX response may contain:

```text
server: Caddy
server: nginx
location: /devices.php
```

Seeing both server headers is possible because Caddy is proxying to the nginx server inside NetAlertX.

## 8. Browser Secure DNS

Chrome and some other browsers can use DNS-over-HTTPS instead of the Mac's system DNS. This can bypass Pi-hole.

In Chrome:

1. Open **Settings**
2. Select **Privacy and security**
3. Open **Security**
4. Find **Use secure DNS**
5. Set it to the current system provider or disable it temporarily
6. Fully quit and reopen Chrome

If terminal resolution works but only the browser fails, check Secure DNS first.

## 9. Caddy internal certificate

The following configuration uses Caddy's private certificate authority:

```caddyfile
tls internal
```

The site may work but display a certificate warning until the Caddy root certificate is trusted on the Mac.

A certificate warning means DNS and routing are already working. It is a different problem from:

```text
Server not found
```

or:

```text
Unknown host
```

The Caddy root certificate is commonly stored on the Caddy server under:

```text
/var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt
```

Import that certificate into macOS Keychain only if you trust and administer the Caddy server.

## 10. Symptom guide

| Symptom | Likely cause |
|---|---|
| Direct Pi-hole `dig` fails | Missing or incorrect Pi-hole record |
| Direct `dig` works but normal `dig` fails | Mac is using another DNS server |
| Both `dig` commands work but `ping` fails | macOS resolver routing or cached state |
| Browser says server not found | DNS resolution or browser Secure DNS |
| `curl --resolve` works | Caddy works; DNS is the problem |
| Connection refused | Caddy is stopped or not listening |
| Caddy returns `502 Bad Gateway` | Caddy cannot reach the upstream application |
| Certificate warning | Caddy internal CA is not trusted |
| Redirect to `/devices.php` | Normal NetAlertX behaviour |

## 11. Recommended diagnostic sequence

Run these on the Mac in order:

```bash
dig +short netalertx.home @192.168.178.26
dig +short netalertx.home
dscacheutil -q host -a name netalertx.home
ping netalertx.home
curl -vkI --resolve netalertx.home:443:192.168.178.108 https://netalertx.home/
```

Then run these on the Caddy server:

```bash
curl -I http://127.0.0.1:20211/
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl status caddy --no-pager
sudo journalctl -u caddy -n 50 --no-pager
sudo ss -lntp | grep -E ':(80|443|20211)\b'
```

## Working fix used in this case

Pi-hole correctly resolved the hostname, but macOS applications still returned `Unknown host`.

The successful fix was:

```bash
sudo mkdir -p /etc/resolver
printf 'nameserver 192.168.178.26\n' | sudo tee /etc/resolver/home
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

After that, macOS sent `.home` hostname queries to Pi-hole and `netalertx.home` became reachable through Caddy.
