# OpenClaw + Open WebUI Docker Stack

A low-config Docker Compose setup for spinning up an OpenClaw agent with an Open WebUI frontend. Pre-configured for Openrouter's current best free model, StepFun Step 3.5. Add whatever models you like!

The only port this exposes publicly is the web interface port for Open-WebUI, which you can configure. The gateway is not exposed publicly, and is totally sandboxed in its container (like a good little lobster).

## What This Is

This stack gives you a **fully functional AI agent** in under two minutes completely for free (preconfigured to use Openrouter with your API key). The stack uses three containers:

- **OpenClaw Gateway** — The agent runtime with tool access, memory, and model routing
- **Open WebUI** — A ChatGPT-like frontend for chatting with your agent
- **Browser Container** — For agents that need web browsing capabilities

**Key features:**
- Single `docker compose up -d` to launch
- One external port (3000)
- Pre-paired authentication between gateway and UI
- Works with any OpenAI-compatible API (Ollama Cloud, local Ollama, etc.)

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Your Browser → http://YOUR-URL-OR-IP:YOUR-PORT         │
│                    │                                    │
│                Open WebUI                               │
│                    │                                    │
│         ┌──────────┴──────────┐                         │
│         │   Shared Secret     │                         │
│         │   (open-handshake)  │                         │
│         └──────────┬──────────┘                         │
│                    │                                    │
│           OpenClaw Gateway                              │
│           (internal:18789)                              │
│                    │                                    │
│         ┌──────────┴──────────┐                         │
│         │  Model Providers    │                         │
│         │  (Ollama Cloud API) │                         │
│         └─────────────────────┘                         │
└─────────────────────────────────────────────────────────┘
```

## Why This Architecture?

During development, we evaluated several approaches:

1. **Fat container (all-in-one)** — Easier to deploy, but harder to update individual components
2. **Separate containers with manual pairing** — Flexible, but required manual token generation
3. **Two-container with pre-shared secret** — Best balance: fast spin-up, clear separation, no manual steps

We landed on **option 3**: Two containers (gateway + webui) on a shared Docker network, with a pre-configured shared secret for instant pairing. The browser container is optional but included for agents that need web access.

## Prerequisites

- Docker and Docker Compose installed
- An API key for a compute provide (Openrouter is free)

## Quick Start

### 1. Clone and Configure

```bash
git clone https://github.com/willificent/openclaw-webui-docker.git
cd openclaw-webui-docker
```

### (Optional) Set the ownership for openclaw folder
If you are using this container in a secure location where the files are owned by anyone other than your user (for example, if you used `sudo` and the files are owned by `root`), the gateway will fail to start. Change the ownship:
```bash
mkdir -p ~/.openclaw
sudo chown -R 1000:1000 ~/.openclaw
```

### (Optional, but suggested) Edit `openclaw/openclaw.json`

Customize the configuration before starting:

```bash
cp openclaw.json.example openclaw.json
# Edit openclaw.json
```
**Important:** The `gateway.auth.token` must match the `OPENAI_API_KEY` in the `open-webui` service environment. The docker-compose.yml uses `openaisharedkey` as a placeholder — change both if you want a different secret.

### (Optional) Set the enviroment variables before launch
There are a few common environment variable that can be set by using an .env files in the Docker Compose root directory:

```bash
cp .env.example .env
# Edit .env
```

### 3. Launch

```bash
docker compose up -d
```

### 4. Setup Open-WebUI
1. Open http://localhost:3000 in your browser (or whatever port you decided on)

2. Create a login

3. Click the circle in the top right, and click "Admin Panel"
<img width="1120" height="318" alt="image" src="https://github.com/user-attachments/assets/101a82ce-412d-491f-aad2-84f930b2cee4" />

4. Click "Settings"
<img width="458" height="166" alt="image" src="https://github.com/user-attachments/assets/dbd33262-5f5c-403c-8ab5-c69656226b31" />

5. Click "Connections"
<img width="341" height="121" alt="image" src="https://github.com/user-attachments/assets/5d6a5b53-e671-4679-9e9e-da1e2f2c738e" />

6. Click the tiny cog on the right to change the OpenAI API Settings:
<img width="800" height="127" alt="image" src="https://github.com/user-attachments/assets/80b8f2c1-252d-47b5-84cb-3452b1a4f2a5" />

7. In the "Add a model ID," type in `agent:main` then click the +
<img width="714" height="766" alt="image" src="https://github.com/user-attachments/assets/03eb703a-4bd3-49f2-a014-97fbd5843257" />

8. Click Save.

---

## 🛠️ Step-by-Step UI Verification

If you are a first-time user, follow these exact steps to ensure the "Handshake" is active:

1. **Check the Gateway Logs:** Run `docker compose logs -f openclaw`. Look for `[info] gateway started on port 18789`.
2. **Verify Connections in Open WebUI:**
   - Click your profile icon (bottom left) → **Settings**.
   - Select **Connections**.
   - Under **OpenAI API**, ensure the status is green.
   - Click the "Refresh" (recycle) icon next to the URL—it should pull the models and show **`agent:main`** in the list.
3. **Set the Default:** You can set `agent:main` as your default model in **Settings > General** so it's always ready when you open the lab.

## Configuration Details

### openclaw.json

This is the main OpenClaw configuration. Key sections:

**Models:**
```json
"providers": {
  "ollama-cloud": {
    "baseUrl": "https://ollama.com/v1",
    "apiKey": "{YOUR-API-KEY}",
    "api": "openai-completions",
    "models": [...]
  }
}
```

**Agents:**
```json
"list": [
  {
    "id": "main",
    "name": "Generic Agent",
    "workspace": "/data/workspace"
  }
]
```

The agent ID `main` appears as `agent:main` in Open WebUI's model selector.

### docker-compose.yml

**Environment variables:**

| Variable | Purpose | Default |
|----------|---------|---------|
| `OPENCLAW_GATEWAY_TOKEN` | Auth token for gateway | `open-handshake-2026` |
| `OPENAI_API_KEY` | Must match gateway token | `open-handshake-2026` |
| `WEBUI_AUTH` | Disable login screen | `false` |
| `WEBUI_NAME` | Display name in UI | `Generic Openclaw Lab` |

**Ports:**

| External | Internal | Service |
|----------|----------|---------|
| 3000 | 8080 | Open WebUI |
| N/A | 18789 | OpenClaw Gateway (external API) |

## Customization

### Using a different provider

Edit `openclaw/openclaw.json`(this example is for Ollama):

```json
"providers": {
  "ollama": {
    "baseUrl": "YOUR-OLLAMA-IP-ADDRESS:11434",
    "api": "ollama",
    "models": [
      { "id": "YOUR-OLLAMA-MODEL", "name": "OLLAMA MODEL NAME" }
    ]
  }
}
```
### Adding More Agents

Add entries to `agents.list`:

```json
{
  "id": "research",
  "name": "Research Agent",
  "workspace": "/data/workspace",
  "model": { "primary": "ollama-cloud/glm-5:cloud" }
}
```
Each agent appears as `agent:{id}` in Open WebUI. If it doesn't appear, add it manually by using the same process as setup (Admin Panel --> Settings --> Connections --> Cog)

### Persistent Storage

Data is stored in the Compose directory by default.

## Security Notes

⚠️ **This setup is designed for local use!**
Be very careful with Openclaw. It can *do* things. Make sure you don't expose ports publicly that you aren't confident are fully secured. You've been warned.

## Files

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Container orchestration |
| `openclaw.json.example` | Template configuration (copy to `openclaw.json`) |
| `openclaw/openclaw.json` | Your customized config (gitignored) |
| `.env` | Your customized environment variables (gitignored) |

## Troubleshooting

**Agent not appearing in Open WebUI:**
- Check gateway logs: `docker compose logs openclaw`
- Verify the token matches between `openclaw.json` and `docker-compose.yml`
- Ensure gateway health check passes: `curl http://localhost:18789/healthz`

**API errors:**
- Verify your API key is valid
- Ensure `baseUrl` ends with `/v1`

**Container keeps restarting:**
- Check for port conflicts (3000, 18789)
- Verify Docker has enough memory (2GB+ recommended)

## Credits
Uses material and inspiration fromn the following sources:
Docker-Openclaw (https://github.com/ideabosque/docker-openclaw/tree/main)
Alpine-Openclaw (https://hub.docker.com/r/alpine/openclaw)
Open WebUI (https://github.com/open-webui/open-webui)
Openclaw (https://github.com/openclaw/openclaw)

Created by [William](https://github.com/willificent) as part of the OpenClaw ecosystem.

## License

MIT — I don't own any of the containers, so do whatever you want with it. Just don't ship your API keys.
