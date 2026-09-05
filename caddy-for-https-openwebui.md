# Using Caddy to Add HTTPS to Open WebUI

A local Open WebUI installation will normally work over HTTP:

```text
http://SERVER_IP:8080
```

However, some browser features require the page to run in a **secure context**.

A common example with Open WebUI is microphone access for voice input. You may encounter:

```text
Permission denied when accessing media devices
```

even though microphone permissions appear to be enabled.

Putting **Caddy** in front of Open WebUI solves this by providing HTTPS:

```text
https://openwebui.home
```

instead of:

```text
http://SERVER_IP:8080
```

This also gives you a friendly local hostname and a reusable HTTPS setup for other self-hosted applications.

---

# 1. Why Does Open WebUI Need HTTPS?

Open WebUI does not require HTTPS simply to send prompts and receive responses.

A basic installation can work like this:

```text
Browser
   |
   | HTTP
   v
Open WebUI :8080
```

However, modern browsers restrict access to security-sensitive APIs when a page isn't considered a secure context.

These can include:

* Microphone access
* Camera access
* Media devices
* Some clipboard functionality
* Other browser APIs requiring a secure context

This matters particularly when using Open WebUI's voice features from a Mac, iPhone or iPad.

The goal is therefore to change:

```text
http://SERVER_IP:8080
```

into:

```text
https://openwebui.home
```

---

# 2. What Caddy Does

Caddy sits between your devices and Open WebUI.

Without Caddy:

```text
Mac / iPhone
      |
      | HTTP :8080
      v
  Open WebUI
```

With Caddy:

```text
Mac / iPhone
      |
      | HTTPS :443
      v
     Caddy
      |
      | HTTP :8080
      v
  Open WebUI
```

Caddy handles HTTPS and certificates while Open WebUI continues to run over ordinary HTTP internally.

---

# 3. Install Caddy

Install Caddy using the appropriate package for your Linux distribution.

Check the service:

```bash
sudo systemctl status caddy
```

Start it if necessary:

```bash
sudo systemctl start caddy
```

Enable it at boot:

```bash
sudo systemctl enable caddy
```

The configuration file is normally:

```text
/etc/caddy/Caddyfile
```

---

# 4. Check Open WebUI First

Before introducing Caddy, make sure Open WebUI works:

```bash
docker compose ps
```

Then:

```bash
curl -I http://127.0.0.1:8080
```

You should also be able to access:

```text
http://SERVER_IP:8080
```

There is little point troubleshooting HTTPS until the underlying Open WebUI service is working.

---

# 5. Create a Local Hostname

Give Open WebUI a local hostname:

```text
openwebui.home
```

Configure local DNS so that:

```text
openwebui.home
```

resolves to the machine running Caddy.

This can be done using:

* Pi-hole
* AdGuard Home
* Your router's local DNS
* Another local DNS server
* A hosts file on desktop operating systems

Check it from Linux or macOS:

```bash
ping openwebui.home
```

or:

```bash
nslookup openwebui.home
```

The hostname should resolve to the **Caddy server**, because Caddy is the service accepting HTTPS connections.

## An important consideration for iPhone and iPad

If you want Open WebUI to work from an iPhone or iPad, configuring DNS centrally is preferable to editing hosts files on individual computers.

For example:

```text
                    Local DNS
                        |
             openwebui.home → server
                    /       \
                   /         \
                 Mac        iPhone
```

Both devices then resolve the same hostname automatically while connected to your home network.

---

# 6. Configure Caddy

Edit:

```bash
sudo nano /etc/caddy/Caddyfile
```

If Caddy and Open WebUI run on the same machine:

```caddy
openwebui.home {
    reverse_proxy 127.0.0.1:8080
    tls internal
}
```

If Open WebUI runs elsewhere:

```caddy
openwebui.home {
    reverse_proxy OPENWEBUI_SERVER_IP:8080
    tls internal
}
```

The two important directives are:

```caddy
reverse_proxy 127.0.0.1:8080
```

and:

```caddy
tls internal
```

The first forwards requests to Open WebUI.

The second tells Caddy to issue the HTTPS certificate using its own internal Certificate Authority.

---

# 7. Why Use `tls internal`?

For a public hostname such as:

```text
chat.example.com
```

Caddy can normally obtain a publicly trusted HTTPS certificate automatically.

A private hostname such as:

```text
openwebui.home
```

cannot normally receive a publicly trusted certificate.

Instead:

```caddy
tls internal
```

makes Caddy act as a private Certificate Authority.

The certificate hierarchy looks roughly like:

```text
Caddy Root CA
      |
      v
Caddy Intermediate CA
      |
      v
openwebui.home
```

The connection is encrypted, but Macs, iPhones and other devices initially don't trust this private CA.

We therefore need to install Caddy's **root certificate** on each client device.

---

# 8. Validate and Reload Caddy

Validate:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

Optionally format the configuration:

```bash
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
```

Reload:

```bash
sudo systemctl reload caddy
```

Test:

```bash
curl -vk https://openwebui.home
```

The `-k` option deliberately ignores certificate trust errors at this stage.

---

# 9. Find Caddy's Root Certificate

On the Caddy server:

```bash
sudo find /var/lib/caddy -name root.crt -print
```

A typical location is:

```text
/var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt
```

This is the certificate that needs to be installed on client devices.

Do **not** install the individual certificate issued for:

```text
openwebui.home
```

Install the **Caddy root CA** instead.

Copy it somewhere accessible:

```bash
sudo cp \
  /var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt \
  /tmp/caddy-root.crt
```

Make it readable:

```bash
sudo chown $USER:$USER /tmp/caddy-root.crt
```

Verify it:

```bash
openssl x509 \
  -in /tmp/caddy-root.crt \
  -noout \
  -subject \
  -issuer
```

---

# 10. Trust Caddy on macOS

Copy the certificate to the Mac:

```bash
scp USER@CADDY_SERVER:/tmp/caddy-root.crt \
  ~/Downloads/caddy-root.crt
```

Install it into the macOS System Keychain:

```bash
sudo security add-trusted-cert \
  -d \
  -r trustRoot \
  -k /Library/Keychains/System.keychain \
  ~/Downloads/caddy-root.crt
```

Verify:

```bash
security find-certificate \
  -a \
  -c "Caddy Local Authority" \
  /Library/Keychains/System.keychain
```

Completely quit Chrome:

```text
Cmd + Q
```

Reopen it and visit:

```text
https://openwebui.home
```

There should no longer be a certificate warning.

---

# 11. Trust Caddy on iPhone and iPad

The same Caddy root certificate can be installed on an iPhone or iPad.

There are **two separate steps** on iOS/iPadOS:

1. Install the certificate profile.
2. Explicitly enable full trust for the root certificate.

Doing only the first step is not enough.

## Get the certificate onto the iPhone

First copy:

```text
caddy-root.crt
```

to the iPhone.

Several methods work, including:

* AirDrop from a Mac
* iCloud Drive
* Emailing it to yourself
* Hosting it temporarily somewhere accessible on your LAN

AirDrop is particularly convenient.

On the Mac, for example:

```bash
scp USER@CADDY_SERVER:/tmp/caddy-root.crt \
  ~/Downloads/caddy-root.crt
```

Then AirDrop `caddy-root.crt` to the iPhone.

## Install the profile

Open the certificate on the iPhone.

iOS should report that a profile has been downloaded.

Go to:

```text
Settings
    → General
    → VPN & Device Management
```

You should see the downloaded certificate/profile.

Select it and tap:

```text
Install
```

Enter the device passcode if requested and complete the installation.

At this point the certificate is installed — but it may **not yet be trusted as a root CA**.

---

# 12. Enable Full Trust on iPhone

Now go to:

```text
Settings
    → General
    → About
    → Certificate Trust Settings
```

Under:

```text
Enable Full Trust for Root Certificates
```

find the Caddy certificate, typically identified as something similar to:

```text
Caddy Local Authority
```

Enable it.

iOS will display a warning explaining that enabling the certificate allows it to establish trusted HTTPS connections.

Confirm it.

This second step is essential.

The iPhone now trusts certificates issued by your Caddy internal CA.

---

# 13. Test Open WebUI on iPhone

Connect the iPhone to the same local network and open Safari.

Visit:

```text
https://openwebui.home
```

It should load without a certificate warning.

If it does, the path is now:

```text
iPhone
   |
   | local DNS
   v
openwebui.home
   |
   | trusted HTTPS :443
   v
 Caddy
   |
   | HTTP :8080
   v
Open WebUI
```

You now have genuine trusted HTTPS rather than merely bypassing a browser warning.

---

# 14. Microphone Access on iPhone

Once HTTPS is trusted, Open WebUI's microphone functionality should be able to request access normally.

When Safari asks whether Open WebUI can access the microphone, choose:

```text
Allow
```

If you previously denied access, check the website's Safari permissions.

While viewing the site in Safari, use the page/site settings and check the microphone permission for:

```text
openwebui.home
```

Set it to:

```text
Allow
```

You can also review Safari's permissions in iOS Settings.

The important sequence is:

```text
Local DNS works
       ↓
HTTPS works
       ↓
Caddy root CA trusted
       ↓
Safari sees a secure context
       ↓
Microphone permission allowed
       ↓
Open WebUI voice input
```

If the certificate still produces a warning, fix that **before** troubleshooting microphone permissions.

---

# 15. `ERR_CERT_AUTHORITY_INVALID`

On desktop Chrome you may initially see:

```text
NET::ERR_CERT_AUTHORITY_INVALID
```

Safari on iPhone may similarly warn that it cannot verify the server's identity.

This normally means:

```text
Caddy is serving HTTPS
          +
The certificate is valid for the hostname
          +
The client doesn't trust Caddy's CA
```

On macOS, install the Caddy root certificate into the System Keychain.

On iPhone/iPad, make sure you have done **both**:

```text
Settings
 → General
 → VPN & Device Management
 → Install certificate
```

and:

```text
Settings
 → General
 → About
 → Certificate Trust Settings
 → Enable Full Trust
```

---

# 16. `ERR_SSL_PROTOCOL_ERROR`

This is different from an untrusted certificate.

If you receive:

```text
ERR_SSL_PROTOCOL_ERROR
```

TLS itself may not be working correctly.

Test:

```bash
curl -vk https://openwebui.home
```

Check Caddy:

```bash
sudo journalctl -u caddy -n 100 --no-pager
```

Validate:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

Also check that the hostname in the Caddyfile exactly matches the hostname you're using.

For example:

```caddy
openwebui.local {
    ...
}
```

doesn't configure:

```text
https://openwebui.home
```

Use:

```caddy
openwebui.home {
    reverse_proxy 127.0.0.1:8080
    tls internal
}
```

---

# 17. Why Not Just Ignore the Certificate Warning?

A browser may allow you to bypass a certificate warning and continue to the site.

That's useful for testing, but it isn't the same as trusting Caddy's CA.

The goal is:

```text
HTTPS
  +
Valid hostname
  +
Trusted CA
  +
Secure browser context
```

This is particularly important when the whole reason for enabling HTTPS is to use security-sensitive browser functionality such as microphones and cameras.

---

# 18. Use Caddy for Other Local Services

Once the Caddy root CA has been trusted on your Mac and iPhone, the same CA can secure other local services.

For example:

```caddy
openwebui.home {
    reverse_proxy 127.0.0.1:8080
    tls internal
}

immich.home {
    reverse_proxy 127.0.0.1:2283
    tls internal
}

grafana.home {
    reverse_proxy 127.0.0.1:3000
    tls internal
}

comfy.home {
    reverse_proxy 127.0.0.1:8188
    tls internal
}
```

Configure local DNS for each hostname:

```text
openwebui.home
immich.home
grafana.home
comfy.home
```

all pointing to Caddy.

The architecture becomes:

```text
 Mac ────────┐
             │
 iPhone ─────┼──── HTTPS ──── Caddy
             │                  │
 iPad ───────┘                  ├── Open WebUI :8080
                                ├── Immich :2283
                                ├── Grafana :3000
                                └── ComfyUI :8188
```

You only need to install and trust the Caddy root CA once on each device.

After that, certificates Caddy issues from the same internal CA for your other local services can also be trusted.

---

# 19. Useful Caddy Commands

Check status:

```bash
sudo systemctl status caddy
```

Validate configuration:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

Format:

```bash
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
```

Reload:

```bash
sudo systemctl reload caddy
```

View logs:

```bash
sudo journalctl -u caddy -n 100 --no-pager
```

Follow logs:

```bash
sudo journalctl -u caddy -f
```

Test HTTPS:

```bash
curl -vk https://openwebui.home
```

Check DNS:

```bash
nslookup openwebui.home
```

Find Caddy's root certificate:

```bash
sudo find /var/lib/caddy -name root.crt -print
```

---

# Final Setup

The finished setup is:

```text
             Local DNS
                 |
                 v
          openwebui.home
                 |
          ┌──────┴──────┐
          │             │
         Mac          iPhone
          │             │
          └──── HTTPS ──┘
                 |
                 v
              Caddy
                 |
                 | HTTP :8080
                 v
            Open WebUI
                 |
                 v
              Ollama
```

Caddy provides HTTPS and issues certificates using its internal Certificate Authority.

The Caddy root CA is then explicitly trusted by:

```text
macOS
  → System Keychain

iPhone / iPad
  → Install Profile
  → Certificate Trust Settings
  → Enable Full Trust
```

This gives Open WebUI a friendly local hostname, encrypted and trusted HTTPS, and a proper secure browser context for functionality such as microphone and voice input.

It also turns Caddy into a useful central HTTPS gateway for the rest of your self-hosted services.
