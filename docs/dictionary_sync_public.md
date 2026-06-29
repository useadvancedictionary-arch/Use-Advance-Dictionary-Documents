# Dictionary sync over the internet (other Wi‑Fi / other cities)

The in-app **Multi-device sync** field must point to a URL that **every phone and PC can reach over the internet** (or over your LAN, if everyone is on the same Wi‑Fi).

This repository ships a tiny Node server: `tools/dictionary_sync_server.mjs`.

**Important:** That server keeps data **in memory only**. If the process restarts, the change log resets. For a production deployment with many users, plan to run it under a process manager and accept restarts, or replace it with a small service backed by Redis/SQLite.

## Minimal copy-paste checklists

Pick **one** path. On every device: open **Multi-device sync** → paste **Sync server base URL** (no trailing `/`) → optional **Bearer token** if the server uses `SYNC_TOKEN` → **Save** → **Sync now**. While the app stays open, it also polls about **every 90 seconds**.

### A — Same Wi‑Fi only (LAN, HTTP)

1. On the PC that will host sync (Node.js installed), open a terminal in the **`tools`** folder of this repo (the folder that contains `dictionary_sync_server.mjs`).
2. Run: `node dictionary_sync_server.mjs`  
   Leave this terminal open (or run the process as a service later).
3. On that PC, find its **LAN IPv4** (e.g. Windows: `ipconfig` → Wireless/Ethernet adapter → something like `192.168.1.105`). **Do not** use `127.0.0.1` on phones— that is the phone itself, not your PC.
4. If other devices cannot connect, allow **inbound TCP 8787** on that PC’s firewall (Private network profile).
5. In the app on **each** phone/tablet/PC: **Sync server base URL** = `http://192.168.x.x:8787` (your real IP). **Bearer token** empty unless you started the server with a token.
6. **Save**, then **Sync now** on each device after you add words on one device.

### B — Different Wi‑Fi / mobile data / internet (HTTPS + token)

1. Deploy the sync server so it has a **public `https://…` URL** (Docker on a VPS + Caddy/nginx TLS, or Fly.io / Railway / Render / Cloud Run — see sections below).
2. Set a long random **`SYNC_TOKEN`** (or equivalent env var) on the server. **Never** leave a public URL without auth.
3. In the app on **every** device: **Sync server base URL** = `https://your-host` (no path, no trailing slash). **Bearer token** = the **same** secret as `SYNC_TOKEN`.
4. **Save**, then **Sync now** to verify.

## 1. Same building (LAN)

- Run: `node tools/dictionary_sync_server.mjs`
- On each device, set base URL to `http://<PC-LAN-IP>:8787` (not `127.0.0.1`).

## 2. Different Wi‑Fi / anywhere (public HTTPS)

You need a **public hostname** with **HTTPS** (recommended). The Flutter app will call:

- `GET https://your-host/v1/dictionary/changes?since=…`
- `POST https://your-host/v1/dictionary/mutations`

### Option A — Docker on any VPS

1. Copy or clone this repo onto a Linux VPS (Ubuntu, etc.).
2. From the **`tools`** directory:

   ```bash
   docker build -f dictionary_sync/Dockerfile -t use-dictionary-sync .
   docker run -d --name dict-sync -p 8787:8787 \
     -e SYNC_TOKEN='choose-a-long-random-secret' \
     use-dictionary-sync
   ```

3. Put **Caddy** or **nginx** in front for TLS, proxy `/` to `127.0.0.1:8787`, obtain a Let’s Encrypt certificate.
4. In the app on **every** device, set **Sync server base URL** to `https://dict.example.com` (no trailing slash), and the same **Bearer token** as `SYNC_TOKEN`.

### Option B — Fly.io (example)

1. Install [Fly CLI](https://fly.io/docs/hands-on/install-flyctl/), run `fly auth login`.
2. From `tools/dictionary_sync`, copy `fly.toml.example` to `fly.toml`, set `app = 'your-unique-name'`.
3. `fly deploy` (Fly builds the Dockerfile and exposes HTTPS).
4. Use `https://your-unique-name.fly.dev` as the app sync base URL.

If Fly **free credits are exhausted** or you prefer not to use Fly, use one of the options below instead (same Docker image and `SYNC_TOKEN` idea).

### Option C — Railway (step by step)

Deploy the sync server from this repo’s **`tools`** folder using Docker. Railway gives you **`https://…`** automatically.

#### Before you start

- Push this project to **GitHub** (or GitLab) and connect that repo to Railway.
- Decide your **`SYNC_TOKEN`**: a long random string (e.g. 32+ characters). You will paste the **same** value into the app as **Bearer token**.

#### Monorepo: where is “root”?

The Docker build must run from the directory that contains **`dictionary_sync_server.mjs`** and the folder **`dictionary_sync/`**:

| Repo layout on GitHub | **Root directory** in Railway |
|----------------------|-------------------------------|
| App repo root = `use_advanced_dictionary_app` (this Flutter project) | **`tools`** |
| Parent repo with app in a subfolder | Path to that `tools` folder, e.g. **`use_advanced_dictionary_app/tools`** |

#### Railway steps

1. Go to [railway.app](https://railway.app) → sign in → **New project**.
2. Choose **Deploy from GitHub repo** (authorize Railway to read repos if asked) and select the repository that contains **`tools/dictionary_sync_server.mjs`**.
3. Railway may auto-create a service. If it guesses **Nixpacks** or the wrong builder, open the service → **Settings** (gear) → switch builder to **Dockerfile** (or delete the service and add **New** → **GitHub Repo** again and pick **Docker** when offered).
4. **Settings → Build:**
   - **Root directory:** `tools` or `use_advanced_dictionary_app/tools` (see table above). Must be the folder that directly contains `dictionary_sync_server.mjs`.
   - **Dockerfile path:** `dictionary_sync/Dockerfile` (relative to that root, not an absolute path).
5. **Variables** (same tab or **Variables** in the sidebar): add  
   **`SYNC_TOKEN`** = your long secret (no quotes needed in the UI).  
   Do **not** set `PORT` manually unless Railway docs say so — Railway usually injects **`PORT`** at runtime; the Node server already reads `process.env.PORT`.
6. **Deploy:** trigger a deploy (or push a commit). Watch **Deployments → Build logs** — success should end with the container starting. If the build fails with “file not found” for `dictionary_sync_server.mjs`, the **root directory** is wrong.
7. **Networking / public URL:** open **Settings → Networking** (or **Generate domain**) and create a **public HTTPS domain** (e.g. `something.up.railway.app`). Copy the full URL **`https://…`** (no path, no trailing slash).
8. **Smoke test:** in a browser open `https://YOUR-RAILWAY-HOST/health` — you should see a small JSON response (e.g. ok). If that fails, check deploy logs and that the service is **not** “Crashed” (wrong `PORT` is rare with this image; wrong Dockerfile path is common).
9. In **USE Advance Dictionary** on **every** device: **Multi-device sync** → **Sync server base URL** = `https://YOUR-RAILWAY-HOST` → **Bearer token** = the same string as **`SYNC_TOKEN`** → **Save** → **Sync now**.

#### If sync still fails in the app

- **401 / “did not accept uploads”:** `SYNC_TOKEN` on Railway and **Bearer token** in the app must match exactly (no extra spaces; redeploy after changing Railway variables).
- **404 on `/health`:** wrong public URL or service not running.
- **Build: COPY failed:** root is not the `tools` folder, or Dockerfile path is not `dictionary_sync/Dockerfile`.

#### Cost

Railway’s free trial / credits are account-specific; if credits are exhausted, use **Option D (Render)**, **Option E (Cloudflare Tunnel)**, or a small VPS (**Option A**).

### Option D — Render (step by step)

1. Sign in at [render.com](https://render.com) → **New** → **Web Service**.
2. Connect your repository. **Root directory:** the `tools` folder (same rule as Railway).
3. **Environment:** **Docker**. **Dockerfile path:** `dictionary_sync/Dockerfile`.
4. **Instance type:** Free is OK for testing; free web services may **spin down** when idle (first request after idle can be slow).
5. Add environment variable **`SYNC_TOKEN`** (and keep **`PORT`** if Render sets it automatically — the server reads `PORT`).
6. Deploy; use the **`https://…onrender.com`** URL Render shows as **Sync server base URL**; set **Bearer token** to match `SYNC_TOKEN`.

### Option E — Home PC + Cloudflare Tunnel (HTTPS without a VPS bill)

Use this when you can leave a PC on at home running the sync server, but phones may be on **cellular** or other Wi‑Fi (so they need a public **HTTPS** URL).

**Quick test URL (changes when you restart the tunnel):**

1. On the PC, run the sync server: from the **`tools`** folder, `node dictionary_sync_server.mjs` (optionally set `SYNC_TOKEN` in the environment first).
2. Install [cloudflared](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/).
3. Run: `cloudflared tunnel --url http://127.0.0.1:8787`  
4. Copy the printed **`https://….trycloudflare.com`** URL into the app as **Sync server base URL**. If you set `SYNC_TOKEN` on the Node process, set the same value as **Bearer token**. **Note:** this hostname is **temporary**; for a stable name, use a named tunnel in the [Cloudflare Zero Trust dashboard](https://one.dash.cloudflare.com/) and route a hostname you control to `http://127.0.0.1:8787`.

### Option F — Google Cloud Run (short)

1. Build and push the image from the **`tools`** directory (same Dockerfile as above), or use Cloud Run’s “build from repo” with root = `tools` and Dockerfile `dictionary_sync/Dockerfile`.
2. Deploy the service; set env **`SYNC_TOKEN`**. Cloud Run sets **`PORT`**; the server respects it.
3. Allow **unauthenticated** HTTP invocations only if you rely entirely on **`SYNC_TOKEN`** for auth (recommended pattern: public URL + strong token). Use the service **HTTPS URL** in the app.

## Security

- **Always set `SYNC_TOKEN`** on public deployments.
- Enter the same value in the app as **Bearer token (optional)** in Multi-device sync.
- Do not expose the sync server without authentication on the open internet.

## App settings recap

- **Laptop + phones on same Wi‑Fi:** `http://192.168.x.x:8787` is enough (HTTP on LAN).
- **Users on cellular / other countries:** they must use **`https://…`** to your deployed host; plain `http://` on random networks is often blocked or unsafe.
