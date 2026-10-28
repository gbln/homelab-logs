# How I Gave My Local LLM Internet Access: Configuring SearXNG in a Proxmox LXC Container and an MCP Server for llama.cpp

> *Giving a local LLM real-time internet search access using SearXNG in a Proxmox LXC container, an MCP server, and llama.cpp—completely private and without API costs.*

---

If you've read my previous posts you already know the setup: a Radeon Vega VII running `llama.cpp` on a Proxmox node, serving local LLM inference over the network. It works well, but a local model working from static training data has an obvious blind spot. It doesn't know what happened last week, can't check a current price, and can't look something up online. The fix is giving it access to a real search engine, privately, without sending queries to Google or paying for an external API.

In this post, I cover exactly that: deploying SearXNG in a Proxmox LXC container, connecting it to `llama-server` through a Model Context Protocol (MCP) server, and getting the whole stack talking to each other. Nine problems documented, nine fixes included.

Here's the full architecture before diving in:

```text
Browser → llama-server (port 8080)
                ↓
          supergateway (port 8081)
                ↓
           mcp-searxng
                ↓
      SearXNG LXC (192.168.68.53:80)
                ↓
             Internet
```

---

## Part 1: SearXNG LXC Container in Proxmox

### Container Specs

| Field | Value |
| :--- | :--- |
| **Template** | Debian 12 (`debian-12-standard_12.12-1_amd64.tar.zst`) |
| **Hostname** | `searxng` |
| **CPU** | 1 core |
| **RAM** | 1 GB |
| **Disk** | 4 GB |
| **IP** | `192.168.68.53/24` |
| **Unprivileged** | Yes |

Create the container through the Proxmox UI or with `pct create`. Once it's running, open a shell inside it.

---

### Problem 1: DNS Not Resolving Inside the Container

Right after the first `apt update`, `curl` couldn't resolve any domains:

```text
curl: (6) Could not resolve host: tailscale.com
```

`ping 8.8.8.8` worked fine—connectivity was there, just no DNS resolution. 

**Fix:** Set explicit nameservers in `/etc/resolv.conf`:

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Now `apt update` works as expected. Continue.

---

### Installing SearXNG

Install system dependencies and prepare the python environment:

```bash
apt update && apt upgrade -y
apt install -y python3 python3-pip python3-venv git curl nginx

# Create a dedicated system user
useradd -r -s /bin/false -m -d /usr/local/searxng searxng

# Clone the source repository
git clone https://github.com/searxng/searxng /usr/local/searxng/searxng-src
chown -R searxng:searxng /usr/local/searxng
```

---

### Problem 2: Missing Python Dependencies

Running `pip install -e .` directly failed with missing module errors:

```text
ModuleNotFoundError: No module named 'msgspec'
ModuleNotFoundError: No module named 'yaml'
```

**Fix:** Instead of chasing dependencies one by one, install everything directly from `requirements.txt` into a dedicated virtual environment:

```bash
cd /usr/local/searxng/searxng-src
su -s /bin/bash searxng -c "python3 -m venv /usr/local/searxng/searxng-pyenv"
su -s /bin/bash searxng -c "/usr/local/searxng/searxng-pyenv/bin/pip install -r /usr/local/searxng/searxng-src/requirements.txt"
su -s /bin/bash searxng -c "/usr/local/searxng/searxng-pyenv/bin/pip install -e ."
```

---

### Configuring SearXNG

Set up the base configuration directory:

```bash
mkdir -p /etc/searxng
cp /usr/local/searxng/searxng-src/searx/settings.yml /etc/searxng/settings.yml
chown -R searxng:searxng /etc/searxng
```

Edit `/etc/searxng/settings.yml` and make sure these specific options are defined:

```yaml
server:
  bind_address: "127.0.0.1"
  port: 8888
  secret_key: "your-key-here"
  limiter: false
  public_instance: false

search:
  formats:
    - html
    - json
```

> **Note:** Generate a proper secret key using `openssl rand -hex 32`.

---

### Problem 3: SearXNG Returning 403 on API Calls

When `mcp-searxng` tried to query SearXNG, it threw a forbidden error:

```text
SearXNG server Error (403): Authentication required or IP blocked
```

By default, SearXNG only serves HTML to browser clients. The MCP server requires structured JSON responses. 

**Fix:** Ensure the `search.formats` block in `/etc/searxng/settings.yml` explicitly includes `- json` and set `limiter: false`. Without both settings, automated API calls will fail.

---

### Creating the systemd Service

Create the service file `/etc/systemd/system/searxng.service`:

```ini
[Unit]
Description=SearXNG service
After=network.target

[Service]
Type=simple
User=searxng
Group=searxng
WorkingDirectory=/usr/local/searxng/searxng-src
Environment="SEARXNG_SETTINGS_PATH=/etc/searxng/settings.yml"
ExecStart=/usr/local/searxng/searxng-pyenv/bin/python -m searx.webapp
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Reload systemd daemon, enable, and start the service:

```bash
systemctl daemon-reload
systemctl enable searxng
systemctl start searxng
```

---

### Nginx as Reverse Proxy

Create the configuration `/etc/nginx/sites-available/searxng`:

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:8888;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Link the configuration and restart Nginx:

```bash
ln -s /etc/nginx/sites-available/searxng /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t
systemctl restart nginx
```

SearXNG is now accessible at `http://192.168.68.53`. Open it in a web browser to confirm it's working properly before moving forward.

---

### Tailscale Inside the Unprivileged LXC

#### Problem 4: Tailscale Failing in an Unprivileged Container

Tailscale requires access to the Linux kernel's TUN device, which is restricted in unprivileged Proxmox LXC containers. The error logged was:

```text
CreateTUN("tailscale0") failed; /dev/net/tun does not exist
```

**Fix:** Apply the fix on the **Proxmox host**, not inside the LXC container. Edit `/etc/pve/lxc/105.conf` (replace `105` with your actual container ID):

```text
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Restart the container, then proceed with the standard Tailscale installation inside the LXC:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
systemctl start tailscaled
systemctl enable tailscaled
tailscale up
```

---

## Part 2: Setting Up the MCP Server on the AI Node

Everything in this section runs on the AI inference host machine, not inside the SearXNG LXC container.

---

### Problem 5: Node.js Version Too Old

Ubuntu 24.04 ships with Node.js v18 by default, which causes syntax and execution errors with `mcp-searxng`:

```text
ReferenceError: File is not defined
```

**Fix:** Upgrade to Node.js v20+ using the official NodeSource repository:

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

Verify with `node --version`. It should output `v20.x.x` or higher.

---

### Installing mcp-searxng and supergateway

Install both packages globally:

```bash
sudo npm install -g mcp-searxng --unsafe-perm
sudo npm install -g supergateway --unsafe-perm
```

---

### Why Supergateway and Not mcpo?

The initial setup attempt used `mcpo` as the MCP proxy wrapper, which led to two critical issues:

* **Problem 6:** `mcpo` lacks support for Server-Sent Events (SSE) and streamable HTTP transport protocols. The web interface of `llama-server` relies on these transports to communicate with external MCP tools.
* **Problem 7:** Even after switching to `supergateway`, connections failed because `supergateway` defaults to standard SSE transport, whereas the modern `llama-server` web interface requires **streamable HTTP**.

**Fix:** Explicitly define the `--outputTransport streamableHttp` flag when launching `supergateway`. Without this flag, tool calls will fail silently despite showing as connected.

---

### Launching the MCP Server

Use a `tmux` session to ensure the background process stays alive after disconnecting SSH:

```bash
tmux new -s mcp
```

Inside the `tmux` session, execute:

```bash
SEARXNG_URL=http://192.168.68.53 supergateway \
  --port 8081 \
  --cors \
  --outputTransport streamableHttp \
  --stdio "mcp-searxng"
```

Allow traffic through the local firewall:

```bash
sudo ufw allow 8081/tcp
```

> **Tmux tips:** Press `Ctrl+B` then `D` to detach from the session. Reattach anytime using `tmux attach -t mcp`.

---

### Problem 8: CORS Blocking Browser Requests

After starting `supergateway`, `llama-server` was unable to connect to the MCP endpoint.

**Reason:** The web UI of `llama-server` initiates tool connections from the user's web browser, not directly from the server backend. Without Cross-Origin Resource Sharing (CORS) headers enabled, browsers block these cross-origin HTTP requests.

**Fix:** Ensure the `--cors` flag is present in the `supergateway` startup command.

---

### Problem 9: Incorrect Host IP Binding

During debugging, connection attempts failed repeatedly due to binding the service to an invalid network interface IP address.

**Fix:** Always verify your active internal network interfaces before hardcoding endpoints:

```bash
ip addr | grep 192.168
```

---

### Connecting to llama-server

1. With `supergateway` active, open your `llama-server` web interface.
2. Navigate to **MCP Servers** → **Add New Server**.
3. Input your endpoint URL:
   ```text
   http://YOUR-AI-SERVER-IP:8081/mcp
   ```
4. The first time your model triggers an web search query, the UI will prompt for tool execution permissions. Select **Allow Always**.

To test functionality, ask the model about real-time events, news items, or weather data that falls outside its training dataset cut-off.

---

## Port Reference Summary

| Port | Service | Location / Scope |
| :--- | :--- | :--- |
| **80** | SearXNG (via Nginx) | Inside LXC Container |
| **8080** | `llama-server` | AI Node |
| **8081** | `supergateway` / MCP Endpoint | AI Node |
| **8888** | SearXNG Internal | LXC Loopback (`127.0.0.1`) |

---

## Closing Thoughts

Nine problems, nine fixes, none of them immediately obvious until encountered. The end result is a completely private, local LLM that can perform live internet searches without external API keys, recurring costs, or data telemetry.

This setup is just one layer of the MCP ecosystem. The same `supergateway` setup can expose multiple tool servers simultaneously—such as `mcp-server-fetch` for parsing web links, `mcp-server-time` for context awareness, or `mcp-server-git` for code repositories. Documenting the full multi-tool stack will be covered in upcoming posts!