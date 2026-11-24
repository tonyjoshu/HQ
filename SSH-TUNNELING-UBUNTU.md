# Internet Access for an Offline Server via SSH Reverse SOCKS Proxy

Give an internet-less server (e.g. `rancher` at `192.168.14.38`) internet access through your workstation (e.g. Kali `nuke`) using a single SSH command.

```
[rancher: apt/curl/docker] --> 127.0.0.1:1080 (SOCKS5) ==SSH tunnel==> [Kali] --> Internet
```

## Requirements

- OpenSSH **7.6+** on the client (Kali). Check with `ssh -V`.
- The workstation has internet access; the server can be reached over SSH.
- `AllowTcpForwarding` is not disabled in the server's `/etc/ssh/sshd_config` (it is enabled by default).

---

## 1. Open the tunnel (on Kali)

```bash
ssh -R 1080 root1@192.168.14.38
```

`-R 1080` with no destination after it tells SSH to start a **SOCKS5 proxy on the server** at `127.0.0.1:1080`. Traffic sent to it goes out through Kali.

To keep only the tunnel open, without a shell:

```bash
ssh -N -R 1080 root1@192.168.14.38
```

> ✅ The tunnel is up if the login shows **no** `Warning: remote port forwarding failed for listen port 1080`.

---

## 2. Test it (on the server)

```bash
curl -x socks5h://127.0.0.1:1080 https://ifconfig.me
```

This should print your workstation's public IP.
`socks5h` (with the **h**) makes DNS lookups happen on the Kali side as well, which matters when the server has no working DNS.

---

## 3. Point tools at the proxy (on the server)

### apt

```bash
echo 'Acquire::http::Proxy "socks5h://127.0.0.1:1080";
Acquire::https::Proxy "socks5h://127.0.0.1:1080";' | sudo tee /etc/apt/apt.conf.d/99proxy

sudo apt update
```

Remove the file when you no longer need it: `sudo rm /etc/apt/apt.conf.d/99proxy`

### Shell (curl, scripts, most CLI tools)

```bash
export ALL_PROXY=socks5h://127.0.0.1:1080
export http_proxy=socks5h://127.0.0.1:1080
export https_proxy=socks5h://127.0.0.1:1080
export no_proxy=localhost,127.0.0.1,192.168.14.0/24
```

> `wget` does not support SOCKS, so use `curl` for downloads.

### Docker (image pulls)

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
printf '[Service]\nEnvironment="HTTPS_PROXY=socks5://127.0.0.1:1080" "HTTP_PROXY=socks5://127.0.0.1:1080" "NO_PROXY=localhost,127.0.0.1,192.168.14.0/24"\n' \
  | sudo tee /etc/systemd/system/docker.service.d/proxy.conf

sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

## Troubleshooting

### `Warning: remote port forwarding failed for listen port 1080`

Something on the server, usually an old SSH session, already holds port 1080.

**Option A: use another port**

```bash
ssh -R 1081 root1@192.168.14.38
# then use 127.0.0.1:1081 everywhere above
```

**Option B: free port 1080 (on the server)**

```bash
# See which process owns the port (sudo is needed to show the PID)
sudo ss -tlnp | grep 1080

# Kill it by PID
sudo kill <pid>

# ...or find and kill it in one step
sudo fuser -k 1080/tcp

# Check: this should print nothing
ss -tnl | grep 1080
```

> ⚠️ The process holding the port is an `sshd` belonging to one of your SSH sessions. Killing it may drop your current connection. Reconnect afterwards.

Then reconnect from Kali with `ssh -R 1080 root1@192.168.14.38`.

### `curl: (97) Recv failure: Connection reset by peer`

The port on the server is a plain forward (e.g. from `-R 1080:127.0.0.1:1080`) to a port on Kali where nothing is listening. Free the port as described above and reconnect using the `-R 1080` form.

### `ssh -D 1080 -N localhost` → `Connection refused`

That method needs an SSH server running on Kali. You don't need it, because `ssh -R 1080` does the same job in one step. If you still want the two-step method:

```bash
sudo systemctl start ssh            # on Kali
ssh -D 1080 -N localhost            # local SOCKS proxy on Kali
ssh -R 1080:127.0.0.1:1080 root1@192.168.14.38   # forward it to the server
```

---

## Notes

- The proxy works **only while the SSH session is open**. Closing it cuts off the server's internet access.
- For a long-running tunnel, run it in `tmux`/`screen`, or use `autossh`:
  ```bash
  autossh -M 0 -N -R 1080 -o ServerAliveInterval=30 -o ServerAliveCountMax=3 root1@192.168.14.38
  ```
- Remove proxy settings (apt file, Docker drop-in, exported variables) once the server has direct internet access.
