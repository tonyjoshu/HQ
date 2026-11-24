# Configure a Reverse SSH Tunnel for Rocky Linux Internet Access

## Overview

A reverse SSH tunnel allows your Rocky Linux machine to access the internet through another computer (your PC) using a SOCKS5 proxy.

This is useful when:
- Your Rocky Linux VM has no direct internet access.
- You need to install packages using `dnf`.
- Your PC has internet access, but your Rocky Linux VM does not.
- You need to download dependencies or install software such as QUADStor VTL.

## 1. Establish the Reverse SSH Tunnel

On your PC (Windows PowerShell, Linux terminal, or macOS terminal), execute:

```bash
ssh -R 1080 rockyvtl@<rocky-ip>
```

Replace `<rocky-ip>` with your Rocky Linux machine's IP address.

**Explanation:**
- `ssh`: Starts an SSH connection.
- `-R`: Creates a remote port forwarding tunnel.
- `1080`: Port where the SOCKS proxy will listen on Rocky Linux.
- `rockyvtl`: Rocky Linux username.

Leave this terminal open while using the tunnel.

**Requirements:**
- OpenSSH version 7.6 or newer.
- SSH connectivity between your PC and Rocky Linux.
- Internet access on your PC.

## 2. Test the SSH Tunnel

On Rocky Linux, execute:

```bash
curl -x socks5h://127.0.0.1:1080 -I https://rockylinux.org
```

If the tunnel works, you should receive an HTTP response such as:

```text
HTTP/2 200
```

or:

```text
HTTP/1.1 301 Moved Permanently
```

This confirms that Rocky Linux can reach the internet through your PC.

## 3. Configure DNF to Use the SOCKS5 Proxy

Configure DNF to use the reverse SSH tunnel:

```bash
echo 'proxy=socks5h://127.0.0.1:1080' | sudo tee -a /etc/dnf/dnf.conf
```

Clear the DNF cache:

```bash
sudo dnf clean all
```

Verify repository connectivity:

```bash
sudo dnf repolist
```

DNF should now use your PC's internet connection through the SOCKS5 proxy.

## 4. Install Required Dependencies

Install the kernel development packages and build tools:

```bash
sudo dnf install -y kernel-devel-$(uname -r) gcc make elfutils-libelf-devel
```

Verify the running kernel and installed kernel development package:

```bash
uname -r
rpm -q kernel-devel
```

Navigate to your QUADStor offline installation directory:

```bash
cd ~/quadstor/quadstor-offline
```

Install the QUADStor VTL RPM package:

```bash
sudo dnf install -y ./quadstor-vtl-std-3.0.79.27-rhel9.x86_64.rpm
```

## 5. Remove the Proxy Configuration

**Important:** When you finish installing the required packages, remove the proxy configuration.

Otherwise, DNF will fail to access repositories whenever the SSH tunnel is disconnected.

Open the DNF configuration:

```bash
sudo nano /etc/dnf/dnf.conf
```

Remove this line:

```ini
proxy=socks5h://127.0.0.1:1080
```

Save and exit.

## 6. Alternative
