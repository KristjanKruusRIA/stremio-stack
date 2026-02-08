# 🚀 Self-Hosted Stremio Stack

A high-performance, secure, and clean self-hosted backend for Stremio.

This stack replaces the traditional Prowlarr setup with **Jackett** for better handling of public indexers, uses **PostgreSQL** for persistent caching, and **Comet** as the high-speed addon. It is designed to be deployed behind a Reverse Proxy (like Caddy) or via Tailscale, keeping all ports closed to the public internet for maximum security.

## ✨ Features

* **⚡ Blazing Fast:** Optimized configuration to serve cached results in <2 seconds.
* **🔒 Secure:** No ports exposed to the host by default. All traffic is routed through an internal Docker network.
* **🧠 Smart Caching:** Uses PostgreSQL to remember search results, making repeated queries instant.
* **🧹 Clean UI:** Configured to filter out raw P2P links and only show "Real-Debrid Cached" results (requires Real-Debrid).
* **🛡️ Captcha Solver:** Includes FlareSolverr to bypass Cloudflare protection on supported indexers.

## 🛠️ Prerequisites

* **Docker** & **Docker Compose** installed.
* **Real-Debrid** API Key (Required for the "Cached Only" experience).
* **(Optional)** A Reverse Proxy (Caddy, Nginx, Traefik) OR Tailscale (for external access).

## 🚀 Deployment

### 1. Clone and Configure
Clone this repository and create your environment file:

```bash
# Copy the example file
cp .env.example .env
Edit the .env file with your preferred settings. Note: You will need to start the stack once to generate the Jackett API Key, then update the .env file and restart.
```

### 2. Start the Stack
```bash
docker-compose up -d
```
### 3. Final Configuration (Critical Steps)
#### A. Jackett Configuration (The Performance Secret)
1. Access Jackett (e.g., http://localhost:9117 or via your proxy).

2. Add Indexers: This stack does not come with pre-configured indexers. You must add them manually.

> ⚠️ Performance Tip: * Do not add 50 indexers blindly. This will slow down your search significantly.
> ⚠️ Pro Tip from the Comet Dev: Avoid adding indexers that strictly require FlareSolverr if you want maximum speed. Cloudflare bypasses can add 10+ seconds to your search time. Recommended fast public indexers: TheRARBG, The Pirate Bay, Knaben. For Anime: Enable native scrapers for Nyaa, AnimeTosho in the Comet config.

3. Test manually: Add a few public indexers and use the "Test" button or perform a manual search.

> ⚠️ The 4-Second Rule: If an indexer takes more than 3-4 seconds to return results, delete it. Comet waits for the slowest indexer. A lean list of fast indexers is better than a huge list of slow ones.

4. Copy API Key: Grab the API Key from the top-right corner of the Jackett dashboard and paste it into your .env file (JACKETT_API_KEY).

5. Restart the stack: docker-compose restart comet.

#### B. Comet Setup
1. Access the Comet configuration page (e.g., http://localhost:8000/configure or your domain).

2. Real-Debrid API Key: Enter your RD token.

3. Max Results: Set to 10 or 15 (prevents infinite scrolling).

4. Show Cached Only: ✅ CHECK THIS. This ensures you only see content that is ready to stream instantly.

5. Deduplicate Streams: ✅ CHECK THIS to keep your list clean.

6. Click Install to add it to Stremio.

## 📺 How to use on Chromecast / Android TV / iOS
Stremio Web and devices like Chromecast often require a valid HTTPS connection to load external addons. Local HTTP addresses (like http://192.168.1.x:8000) often fail or get blocked by Mixed Content policies.

The easiest way to make this work without opening ports on your router is using **Tailscale Serve**.

> **Tailscale Serve vs Funnel:**
> - **Serve** = Private to your Tailnet only (recommended - only your devices can access)
> - **Funnel** = Public internet (anyone with the URL can access)

### Step-by-Step for Tailscale Users:
1. Install Tailscale on your server and all devices that need access (phone, TV, etc.).

2. Run the following command on your server to expose Comet privately to your Tailnet:

```bash
# Expose Comet only to devices on your Tailnet (private)
tailscale serve --bg --https=443 http://localhost:8000
```

3. Verify it's running privately:
```bash
tailscale serve status
```

4. Tailscale will generate a private URL (e.g., `https://my-server.tailnet-name.ts.net`). Use that URL as your `PUBLIC_BASE_URL` in the `.env` file.

5. Restart the stack:
```bash
docker compose down && docker compose up -d
```

6. Configure the addon using the new HTTPS URL. Now it will work on Chromecast, iOS, and Smart TVs — but **only from devices logged into your Tailscale account**.

> **Note:** If you need public access (not recommended), use `tailscale funnel` instead of `tailscale serve` and enable Funnel in your Tailscale Admin Console.

## 🌐 CometNet (P2P Torrent Sharing)

CometNet allows Comet instances to share torrent metadata with each other, improving discovery coverage for all participants. When you find a torrent, it gets shared with the network, and vice versa.

### Enabling CometNet

Add these to your `.env`:

```env
COMETNET_ENABLED=True
FASTAPI_WORKERS=1

# Your public WebSocket URL (required for others to connect to you)
COMETNET_ADVERTISE_URL=wss://your-tailscale-hostname.ts.net:8765
```

### Exposing CometNet Publicly

CometNet uses port 8765 for WebSocket connections. To allow others to connect:

1. **Add port 8765** to your `docker-compose.yml` comet service:
   ```yaml
   ports:
     - "8000:8000"
     - "8765:8765"
   ```

2. **Expose via Tailscale Funnel** (CometNet requires public access):
   ```bash
   tailscale funnel --bg --https=8765 http://localhost:8765
   ```

3. **Restart the stack**:
   ```bash
   docker compose down && docker compose up -d
   ```

### Connecting to Others

To connect to other CometNet nodes, add their URLs to your `.env`:

```env
# Bootstrap nodes (for discovering the network)
COMETNET_BOOTSTRAP_NODES='["wss://bootstrap.example.com:8765"]'

# Or direct peer connections (friends' Comet instances)
COMETNET_MANUAL_PEERS='["wss://friend-server.ts.net:8765"]'
```

### Your CometNet URL

Share this URL with friends who want to connect to your instance:
```
wss://your-tailscale-hostname.ts.net:8765
```

They add it to their `COMETNET_MANUAL_PEERS` in their `.env` file.

### Private Networks (Friends Only)

For a closed group of trusted friends:

```env
COMETNET_PRIVATE_NETWORK=True
COMETNET_NETWORK_ID=my-friends-network
COMETNET_NETWORK_PASSWORD=shared-secret-password
```

All members must use the same Network ID and Password.

### Verify CometNet is Working

Check the container logs for:
```
CometNet started - Node ID: abc123...
Connected to peer def456...
```

Or visit the Admin Dashboard at `http://localhost:8000/admin` → CometNet tab.

## 📂 Project Structure
* docker-compose.yml: Main stack definition.

* .env.example: Template for environment variables.

* Caddyfile: (Optional) Configuration example if you prefer using Caddy instead of Tailscale.

## ⚠️ Notes
* **Security** The ports section in docker-compose.yml is commented out by default. This is intentional. Traffic should flow through the internal Docker network (proxy_net) to your Reverse Proxy or Tailscale. If you need direct IP access, uncomment the ports in the YAML file.

* **Logs**: If searches are slow, check the Jackett logs. Identify which indexer is timing out and remove it.

## 🤝 Contributing
Feel free to fork this repository and submit pull requests to improve the configuration or documentation.