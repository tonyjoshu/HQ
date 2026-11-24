# Configure a Reverse SSH Tunnel for Ubuntu Internet Access

## Overview

A reverse SSH tunnel allows an Ubuntu machine without direct internet access to use another computer's internet connection through a SOCKS5 proxy.

This is useful when:
- Your Ubuntu VM has no internet access.
- Your PC has internet access, but Ubuntu does not.
- You need to download packages using APT.
- You need to install software and dependencies on an offline Ubuntu server.

## 1. Establish the Reverse SSH Tunnel

On your PC (Windows PowerShell, Linux terminal, or macOS terminal), execute:

```bash
ssh -R 1080:127.0.0.1:1080 ubuntuuser@<ubuntu-ip>
```

Replace:
- `ubuntuuser` with your Ubuntu username.
- `<ubuntu-ip>` with your Ubuntu machine's IP address.

Leave this terminal open.

**Important:** The command above forwards a remote port to a local port. To provide an actual SOCKS5 proxy, start a SOCKS proxy on your PC first.

Open another terminal on your PC and execute:

```bash
ssh -D 1080 -N localhost
```

A more practical arrangement is to use a local SOCKS5 proxy application such as `ssh -D`, then establish the reverse forwarding with a separate SSH connection. Ensure the ports do not conflict.

## 2. Test the Tunnel

On Ubuntu, execute:

```bash
curl -x socks5h://127.0.0.1:1080 -I https://ubuntu.com
```

A successful connection should return an HTTP response such as:

```text
HTTP/2 200
```

## 3. Configure APT to Use the SOCKS5 Proxy

APT supports HTTP and HTTPS proxies natively, but not SOCKS5 directly.

Install ProxyChains:

```bash
sudo apt update
sudo apt install -y proxychains4
```

Edit its configuration:

```bash
sudo nano /etc/proxychains4.conf
```

At the bottom, configure:

```ini
[ProxyList]
socks5 127.0.0.1 1080
```

Test connectivity:

```bash
proxychains4 curl -I https://ubuntu.com
```

## 4. Install Packages Through the Proxy

Run APT commands through ProxyChains:

```bash
sudo proxychains4 apt update
```

For package installation:

```bash
sudo proxychains4 apt install -y gcc make
```

**Note:** Running ProxyChains with `sudo` can require additional configuration depending on the system. If APT does not work through ProxyChains, use an HTTP proxy bridge such as Privoxy between SOCKS5 and APT.

## 5. Remove the Proxy Configuration

When finished, edit:

```bash
sudo nano /etc/proxychains4.conf
```

Remove or comment out the SOCKS5 entry if you no longer need it.

## 6. Alternative: Use NAT Networking

If Ubuntu is running inside a virtual machine, NAT networking is usually the simplest option.

NAT allows the VM to share the host computer's internet connection without manually establishing an SSH tunnel.

## Summary

| Component | Purpose |
|---|---|
| SSH | Establishes the secure connection |
| SSH Reverse Forwarding | Exposes a port on Ubuntu |
| SOCKS5 | Forwards network traffic |
| ProxyChains | Routes supported applications through SOCKS5 |
| APT | Installs Ubuntu packages |
| NAT | Alternative VM internet connectivity |

**Final Note:** Keep the SSH tunnel active while downloading packages. Close it when you finish.
