# Installing ntopng on Ubuntu 24.04

This guide installs the current stable version of ntopng from ntop's official repository. It does not use Ubuntu's older `universe` build of ntopng.

ntopng is a web-based traffic-monitoring application. It can show active hosts, network flows, application protocols, upload and download rates, and historical traffic statistics.

> **Important:** Installing ntopng on an ordinary computer does not automatically let it see traffic from every device on the network. The monitored computer must be the network gateway, receive exported flow data, or have a network interface connected to a managed switch's mirror/SPAN port.

## Requirements

- Ubuntu 24.04 LTS on an x86-64 computer
- Administrative access through `sudo`
- A wired network interface is preferable
- For whole-network monitoring, a managed switch supporting port mirroring and preferably a dedicated capture interface

## 1. Confirm the Ubuntu version

```bash
lsb_release -ds
```

This guide expects Ubuntu 24.04 LTS.

## 2. Add the official stable repository

Install the repository prerequisites:

```bash
sudo apt update
sudo apt install software-properties-common wget
sudo add-apt-repository universe
```

Download and install ntop's repository package:

```bash
cd /tmp
wget https://packages.ntop.org/apt-stable/24.04/all/apt-ntop-stable.deb
sudo apt install ./apt-ntop-stable.deb
sudo apt update
```

Confirm that APT can see ntop's package:

```bash
apt-cache policy ntopng
```

The candidate should come from:

```text
https://packages.ntop.org/apt-stable/24.04
```

Do not proceed if the candidate is the old Ubuntu `5.2.1+dfsg1` package.

## 3. Install ntopng

```bash
sudo apt install ntopng
```

Do not use `-y`. The ntop packages can display installation or licensing information that should be reviewed before continuing.

The installation pulls in Redis and the other required dependencies.

## 4. Verify the installation

Clear any old command-location information cached by Bash:

```bash
hash -r
```

Then check the executable and version:

```bash
command -v ntopng
ntopng --version
```

The official package normally installs the executable as:

```text
/usr/bin/ntopng
```

To find the package-owned executable explicitly:

```bash
dpkg -L ntopng | grep -E '/(s)?bin/ntopng$'
```

## 5. Start the services

```bash
sudo systemctl enable --now redis-server
sudo systemctl enable --now ntopng
sudo systemctl status ntopng --no-pager
```

If ntopng does not start, inspect its log:

```bash
sudo journalctl -u ntopng -n 100 --no-pager
```

## 6. Open the web interface

Find the Ubuntu computer's address:

```bash
hostname -I
```

Recent ntop packages enable HTTPS on port `3001` and disable plain HTTP by default. Open:

```text
https://SERVER-IP:3001
```

For example:

```text
https://192.168.178.108:3001
```

The initial certificate is self-signed, so the browser will display a certificate warning. Confirm that the address is your ntopng server before using the browser's advanced option to continue.

The initial credentials are normally:

```text
Username: admin
Password: admin
```

ntopng should require a new administrator password at first login.

Check which web port is listening with:

```bash
sudo ss -lntp | grep -E ':3000|:3001'
```

A normal current installation should show port `3001` listening.

## 7. Allow access through UFW

First check whether UFW is active:

```bash
sudo ufw status
```

If it is active, allow ntopng only from the local network:

```bash
sudo ufw allow from 192.168.178.0/24 to any port 3001 proto tcp
```

Change the subnet if the home network does not use `192.168.178.0/24`.

Do not expose the ntopng interface directly to the internet.

## 8. Identify the network interfaces

```bash
ip -br link
ip -br address
ip route show default
```

The default route identifies the normal LAN interface. For example:

```text
default via 192.168.178.1 dev enp6s0
```

In this example the interface is `enp6s0`.

Avoid selecting Docker or virtual interfaces such as these unless they are intentionally being monitored:

- `docker0`
- `br-*`
- `veth*`
- `lo`

## 9. Configure the monitored interface

The official package normally stores its configuration at:

```text
/etc/ntopng/ntopng.conf
```

Confirm the installed configuration location if necessary:

```bash
dpkg -L ntopng | grep -E 'ntopng\.conf$'
```

Back up the configuration before editing it:

```bash
sudo cp -a /etc/ntopng/ntopng.conf /etc/ntopng/ntopng.conf.backup
```

Edit it:

```bash
sudo nano /etc/ntopng/ntopng.conf
```

Configure the required interface and local network, substituting the correct interface name:

```text
--interface=enp6s0
--local-networks=192.168.178.0/24
```

Configuration-file options require an equals sign between each option and its value.

Restart ntopng after changing the configuration:

```bash
sudo systemctl restart ntopng
sudo systemctl status ntopng --no-pager
```

## 10. Test traffic collection

Open the ntopng dashboard and navigate to the hosts view. Generate some traffic from the Ubuntu machine:

```bash
curl -o /dev/null 'https://speed.cloudflare.com/__down?bytes=100000000'
```

The Ubuntu host's throughput should increase in ntopng.

If ntopng shows only the Ubuntu computer, router, and broadcast traffic, it is working correctly but its interface cannot observe the other devices' packets.

## Whole-network monitoring

On a switched Ethernet network, one computer normally receives only its own unicast traffic. For household-wide monitoring, use a managed switch with port mirroring:

1. Connect the FRITZ!Box LAN uplink to the managed switch.
2. Connect the Deco system and other wired devices through the switch.
3. Configure the FRITZ!Box-facing switch port as the mirror source in both directions.
4. Configure a separate switch port as the mirror destination.
5. Connect the mirror destination to a dedicated Ethernet interface on the ntopng computer.
6. Configure ntopng to monitor that dedicated interface.

The capture interface generally does not require an IP address. Keep a separate normal interface for administering the Ubuntu computer and opening the ntopng dashboard.

If a Deco system is operating in router mode, traffic observed between the Deco and FRIT!Box may appear to originate from the Deco because of NAT. Access Point mode normally preserves individual device addresses and avoids double NAT.

## Useful checks

### Service status

```bash
sudo systemctl status ntopng --no-pager
```

### Recent logs

```bash
sudo journalctl -u ntopng -n 100 --no-pager
```

### Listening ports

```bash
sudo ss -lntp | grep -E ':3000|:3001'
```

### Installed package versions

```bash
apt-cache policy ntopng ntopng-data
```

### Package integrity

```bash
sudo dpkg -V ntopng ntopng-data
```

### Restart ntopng

```bash
sudo systemctl restart ntopng
```

## Troubleshooting

### Bash reports the old `/usr/sbin/ntopng` path

After replacing an older package, Bash may remember its former executable location:

```text
-bash: /usr/sbin/ntopng: No such file or directory
```

Clear the cached path:

```bash
hash -r
command -v ntopng
ntopng --version
```

If the package lists the executable but the file is missing, reinstall the matching packages:

```bash
sudo apt install --reinstall ntopng ntopng-data
```

### Port 3000 does not respond

Current packages normally use HTTPS port `3001`, not HTTP port `3000`:

```text
https://SERVER-IP:3001
```

Confirm with:

```bash
sudo ss -lntp | grep -E ':3000|:3001'
```

### The service repeatedly stops

```bash
sudo journalctl -u ntopng -n 100 --no-pager
sudo systemctl status redis-server --no-pager
sudo systemctl status ntopng --no-pager
```

### The dashboard works but other devices are absent

This is normally a traffic-visibility issue, not an ntopng fault. An ordinary network interface cannot see unrelated switched traffic. Use a gateway installation, flow exporter, network tap, or managed-switch mirror port.

### Device names are missing

ntopng may initially display IP or MAC addresses. Give important devices DHCP reservations and meaningful hostnames. Pi-hole can complement ntopng by showing DNS requests, but it does not carry or measure the devices' application traffic.

## Updating ntopng

Update through APT:

```bash
sudo apt update
apt list --upgradable
sudo apt upgrade
```

Check the version afterwards:

```bash
hash -r
ntopng --version
```

The Community edition does not require a paid licence.

## References

- [ntopng installation documentation](https://www.ntop.org/guides/ntopng/installation.html)
- [ntop stable Ubuntu packages](https://packages.ntop.org/apt-stable/)
- [ntopng documentation](https://www.ntop.org/guides/ntopng/)
- [Mirror/SPAN/TAP monitoring](https://www.ntop.org/guides/ntopng/use_cases/mirror_tap_monitoring.html)
- [ntopng Community edition](https://www.ntop.org/guides/ntopng/versions_and_licensing.html)
