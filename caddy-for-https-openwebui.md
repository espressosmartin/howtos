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

This matters particularly when using Open WebUI's voice features from a Mac, iPhone, iPad or Android device.

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

Caddy sits between your devices and Open WebUI:

```text
Mac / iPhone / Android
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

# 3. Configure Caddy

Assuming Open WebUI is running on the same machine as Caddy:

```caddy
openwebui.home {
    reverse_proxy 127.0.0.1:8080
    tls internal
}
```

If Open WebUI runs on another machine:

```caddy
openwebui.home {
    reverse_proxy OPENWEBUI_SERVER_IP:8080
    tls internal
}
```

The important directive is:

```caddy
tls internal
```

This makes Caddy use its own internal Certificate Authority.

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

The connection is encrypted, but your computers, phones and tablets initially don't trust this private CA.

---

# 4. Configure Local DNS

Configure your local DNS so:

```text
openwebui.home
```

points to the machine running Caddy.

This can be configured centrally using something such as:

* Pi-hole
* AdGuard Home
* Your router
* Another local DNS server

Central DNS is particularly useful for mobile devices because Macs, iPhones, iPads and Android devices can all use the same hostname without individual hosts-file configuration.

Check it with:

```bash
nslookup openwebui.home
```

---

# 5. Validate and Reload Caddy

Validate:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

Optionally format:

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

The `-k` deliberately ignores certificate trust errors while testing.

---

# 6. Find Caddy's Root Certificate

On the Caddy server:

```bash
sudo find /var/lib/caddy -name root.crt -print
```

A typical location is:

```text
/var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt
```

This is the certificate that needs to be trusted by your client devices.

Do **not** install the individual certificate issued for `openwebui.home`. Install Caddy's **root CA**.

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

# 7. Trust Caddy on macOS

Copy the certificate to the Mac:

```bash
scp USER@CADDY_SERVER:/tmp/caddy-root.crt \
  ~/Downloads/caddy-root.crt
```

Install it into the System Keychain:

```bash
sudo security add-trusted-cert \
  -d \
  -r trustRoot \
  -k /Library/Keychains/System.keychain \
  ~/Downloads/caddy-root.crt
```

Completely quit and reopen your browser.

Then visit:

```text
https://openwebui.home
```

There should no longer be a certificate warning.

---

# 8. Trust Caddy on iPhone and iPad

Copy `caddy-root.crt` to the iPhone or iPad. AirDrop from a Mac is one convenient method.

Open the certificate on the device.

Then go to:

```text
Settings
    → General
    → VPN & Device Management
```

Select the downloaded certificate/profile and choose:

```text
Install
```

Installing it isn't quite enough.

Next go to:

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

find the Caddy Local Authority and enable it.

Confirm the warning.

Safari should now trust:

```text
https://openwebui.home
```

---

# 9. Trust Caddy on Android

Android can also trust Caddy's private Certificate Authority.

The exact names of the menus vary between Android versions and manufacturers such as Google, Samsung and OnePlus, but the process is generally:

```text
Copy Caddy root certificate to phone
                ↓
Install it as a CA certificate
                ↓
Android adds it to user credentials
                ↓
Browser can trust certificates issued by it
```

## Get the certificate onto Android

Copy:

```text
caddy-root.crt
```

to the Android device.

You could use:

* USB
* Google Drive or another file-sync service
* A local file share
* Email
* Nearby Share / Quick Share
* A temporary local download

The important thing is that you're transferring the **Caddy root CA**, not the certificate for `openwebui.home`.

## Install the CA certificate

On recent stock Android versions, look for something similar to:

```text
Settings
    → Security & privacy
    → More security settings
    → Encryption & credentials
    → Install a certificate
    → CA certificate
```

On some versions it may instead be:

```text
Settings
    → Security
    → Encryption & credentials
    → Install a certificate
    → CA certificate
```

Samsung devices may use wording such as:

```text
Settings
    → Security and privacy
    → More security settings
    → Install from device storage
    → CA certificate
```

Select:

```text
caddy-root.crt
```

Android will display a security warning because a CA certificate can be used to establish trust for HTTPS connections.

Confirm the installation.

You may need to authenticate using your PIN, password or fingerprint.

## If Android doesn't recognise `.crt`

Some Android versions or manufacturers can be particular about certificate files.

Caddy's root certificate is normally PEM encoded. If Android refuses to import it, convert it to DER format on the Caddy server:

```bash
openssl x509 \
  -in /tmp/caddy-root.crt \
  -outform DER \
  -out /tmp/caddy-root.cer
```

Copy:

```text
caddy-root.cer
```

to the Android device and try the certificate installation again.

---

# 10. Check the Certificate on Android

After installation, Android should list the CA under its user-installed credentials.

Look for something similar to:

```text
Settings
    → Security & privacy
    → More security settings
    → Encryption & credentials
    → Trusted credentials
    → User
```

The exact path varies between devices.

You should see an entry corresponding to Caddy's local Certificate Authority.

This distinction is useful:

```text
System certificates
        |
        └── supplied by Android/device manufacturer

User certificates
        |
        └── Caddy Local Authority
```

Caddy will normally appear as a **user-installed CA**.

---

# 11. Test Open WebUI on Android

Make sure the Android device is connected to the local network and that DNS resolves:

```text
openwebui.home
```

to your Caddy server.

Open:

```text
https://openwebui.home
```

in Chrome.

If everything is working, Open WebUI should load without a certificate warning.

The request path is now:

```text
Android
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

---

# 12. An Important Android Caveat

Android distinguishes between **system-installed** and **user-installed** Certificate Authorities.

Modern Android applications can choose not to trust user-installed CAs.

This means installing the Caddy root certificate does **not automatically guarantee that every Android app will trust it**.

For our Open WebUI use case, we're primarily interested in accessing:

```text
https://openwebui.home
```

through a browser.

If a particular Android browser or application continues to reject the certificate, check whether that application accepts user-installed Certificate Authorities.

Avoid trying to modify Android's system CA store unless you specifically understand the security implications; doing so typically requires device-management capabilities or root access and isn't necessary for a normal Open WebUI setup.

---

# 13. Microphone Access on Android

Once:

```text
https://openwebui.home
```

loads without a certificate warning, try Open WebUI's voice functionality.

Chrome should ask:

```text
Allow openwebui.home to use your microphone?
```

Choose:

```text
Allow
```

If you previously denied it, check Chrome's site permissions.

In Chrome, open:

```text
Settings
    → Site settings
    → Microphone
```

and check the permissions for `openwebui.home`.

Android itself must also allow Chrome to use the microphone.

Look under:

```text
Settings
    → Apps
    → Chrome
    → Permissions
    → Microphone
```

and allow microphone access.

The full chain is therefore:

```text
Local DNS works
       ↓
HTTPS works
       ↓
Caddy root CA trusted
       ↓
Chrome accepts certificate
       ↓
Android allows Chrome microphone
       ↓
Chrome allows site microphone
       ↓
Open WebUI voice input
```

---

# 14. Troubleshooting Certificate Errors

If Chrome reports:

```text
NET::ERR_CERT_AUTHORITY_INVALID
```

then HTTPS may be working but the Caddy CA isn't trusted correctly.

Check that you installed the **root CA**, not the individual `openwebui.home` certificate.

If you get:

```text
ERR_SSL_PROTOCOL_ERROR
```

the problem is more likely with the TLS/Caddy configuration itself.

Check:

```bash
curl -vk https://openwebui.home
```

and:

```bash
sudo journalctl -u caddy -n 100 --no-pager
```

Also check that the Caddy hostname exactly matches:

```caddy
openwebui.home {
    reverse_proxy 127.0.0.1:8080
    tls internal
}
```

---

# 15. One Caddy CA for All Your Devices

Once configured, the same Caddy CA can provide HTTPS across your home network:

```text
                    Local DNS
                        |
                        v
                  openwebui.home
                        |
                        v
                      Caddy
                        |
          ┌─────────────┼─────────────┐
          │             │             │
         Mac          iPhone       Android
          │             │             │
     Trust root     Trust root     Trust root
          │             │             │
          └──────── HTTPS ────────────┘
```

You only need to trust the Caddy root CA **once on each device**.

The same Caddy installation can then secure other local services:

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
```

Once your devices trust the Caddy root CA, certificates Caddy issues for these additional local services can also be trusted.

---

# Final Setup

The finished setup looks like:

```text
                       Local DNS
                           |
                           v
                    openwebui.home
                           |
             ┌─────────────┼─────────────┐
             │             │             │
            Mac          iPhone       Android
             │             │             │
             └──────── HTTPS :443 ───────┘
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

Caddy provides HTTPS using its internal Certificate Authority.

Each client trusts that CA:

```text
macOS
  → System Keychain

iPhone / iPad
  → Install Profile
  → Certificate Trust Settings
  → Enable Full Trust

Android
  → Install CA certificate
  → User trusted credentials
```

The result is a locally hosted Open WebUI installation with a friendly hostname, trusted HTTPS and a secure browser context for functionality such as microphone and voice input.

The same Caddy CA can then be reused to provide HTTPS for other self-hosted services across the local network.
