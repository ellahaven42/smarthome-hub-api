# SmartHome Hub API

Unify any smart device behind one simple, serverless HTTP API.

> Serverless hub API for smart‑home control. **FastAPI** on **AWS Lambda + API Gateway** with token auth, one‑tap scenes, and device actions. Ships direct adapters (Hue/WiZ/LIFX) and an optional **Home Assistant** bridge to reach Alexa, Google, LG/Midea, HomeKit, and more.

---

## Table of Contents

* [About](#about)
* [Features](#features)
* [Architecture](#architecture)
* [Tech Stack](#tech-stack)
* [Project Structure](#project-structure)
* [Getting Started](#getting-started)

  * [Local Development](#local-development)
  * [Manual Deploy (Lambda + API Gateway)](#manual-deploy-lambda--api-gateway)
  * [Deploy with SAM (recommended)](#deploy-with-sam-recommended)
* [Configuration](#configuration)
* [API Reference](#api-reference)
* [Examples](#examples)
* [Security Notes](#security-notes)
* [Roadmap](#roadmap)
* [Contributing](#contributing)
* [License](#license)
* [Contact](#contact)

---

## About

**SmartHome Hub API** is the central “brain” for quick, reliable home automations. It exposes a clean REST interface (scenes, toggles, state updates) powered by **FastAPI** and deployed on **AWS Lambda + API Gateway**. Out of the box it includes **direct adapters** for Philips **Hue**, **WiZ**, and **LIFX**. For everything else—**Alexa**, **Google Home**, **HomeKit**, **LG/Midea ACs**, and countless devices—you can enable the **Home Assistant (HA) bridge**, letting HA act as a universal translator so the same API calls reach any brand without custom vendor code.

Why this repo? It showcases practical API design, serverless deployment, and IoT integration patterns you can extend at home or in a portfolio project.

---

## Features

* **One‑tap scenes** (e.g., `POST /quick/good-night`)
* **Device actions**: toggle, set brightness / color temperature / modes
* **Token auth** via `x-api-key` header (secret in SSM)
* **Serverless**: AWS Lambda + API Gateway + CloudWatch logs
* **Custom domain** support (e.g., `https://api.yourdomain.com`)
* **Adapters** (stubs in Phase 1): Hue, WiZ, LIFX; HA bridge for the long tail

---

## Architecture

```
[iOS Shortcut / Widget / Any Client]
               |
        HTTPS (x-api-key)
               v
     API Gateway (HTTP API)
               |
           AWS Lambda
               |
            FastAPI
        /       \
 Direct Adapters  Home Assistant Bridge
(Hue/WiZ/LIFX)      (REST/WebSocket)
```

**Approach:** Start simple with stubbed handlers → add LAN adapters for Hue/WiZ/LIFX → optionally enable the HA bridge to control everything else without bespoke vendor code.

---

## Tech Stack

* **Backend:** FastAPI (Python 3.11+), Pydantic
* **Serverless:** AWS Lambda, API Gateway, **Mangum** adapter
* **Infra:** Route 53 (DNS), ACM (TLS), CloudWatch (logs) — **AWS SAM** template optional
* **Secrets:** AWS SSM Parameter Store (SecureString)

---

## Project Structure

```
app/
  main.py          # FastAPI app factory + /health
  auth.py          # x-api-key middleware
  api/
    routes.py      # /quick, /scene, /device endpoints
  services/
    scenes.py      # scene runner (stub)
    devices.py     # device actions (stub)
  adapters/        # hue/, wiz/, lifx/, home_assistant/ (later phases)
handler.py         # Lambda entry (Mangum)
requirements.txt   # Python dependencies
README.md
.env.example       # local-only env vars
scripts/
  package_lambda.sh# zip for Lambda upload (manual deploy)
```

---

## Getting Started

### Local Development

1. **Clone & setup**

   ```bash
   git clone https://github.com/<you>/smarthome-hub-api.git
   cd smarthome-hub-api
   python -m venv .venv
   source .venv/bin/activate   # Windows: .venv\Scripts\activate
   python -m pip install --upgrade pip
   python -m pip install -r requirements.txt
   cp .env.example .env && edit .env   # set API_KEY
   ```
2. **Run**

   ```bash
   uvicorn app.main:app --reload --host 127.0.0.1 --port 8080
   ```

   Open docs: [http://127.0.0.1:8080/docs](http://127.0.0.1:8080/docs)
3. **Smoke test**

   ```bash
   curl -i http://127.0.0.1:8080/health
   curl -i -X POST http://127.0.0.1:8080/quick/good-night -H "x-api-key: <your-dev-key>"
   ```

### Manual Deploy (Lambda + API Gateway)

1. **Package**

   ```bash
   bash scripts/package_lambda.sh
   ```
2. **Lambda**: Create function (Python 3.11) → upload `build/lambda.zip`.
3. **API Gateway (HTTP API)**: Create API → integrate Lambda (proxy) → deploy stage.
4. **Env**: Set `API_KEY` (temp) or wire SSM (see below).
5. **Test**: `GET <base-url>/health` returns 200.

### Deploy with SAM (recommended)

> If you add `template.yaml`, you can build & deploy repeatably:

```bash
sam build
sam deploy --guided
```

* Choose a stack name and region (e.g., `us-east-1`).
* Later attach a **custom domain** via API Gateway + ACM + Route 53.

---

## Configuration

Environment variables (use `.env` locally; Lambda env or SSM in prod):

| Name      | Required | Example                    | Notes                                   |
| --------- | -------- | -------------------------- | --------------------------------------- |
| `API_KEY` | Yes      | `super-long-random-string` | Compared to `x-api-key` header          |
| `APP_ENV` | No       | `dev` \| `prod`            | Enables dev conveniences (warn on auth) |

**SSM Parameter Store (prod):**

* Create **SecureString** parameter, e.g. `/smarthome/hub/API_KEY`.
* Grant Lambda role `ssm:GetParameter` (+ `kms:Decrypt` if custom KMS).
* Load at cold start and cache, or inject into env at deploy.

---

## API Reference

### Health

```
GET /health
200 OK
{
  "status": "ok",
  "uptime_seconds": 12.345
}
```

### Quick Scenes

```
POST /quick/{scene}
# Allowed (Phase 1): good-night, wake-gentle, focus, all-off
Headers: x-api-key

200 OK
{
  "ok": true,
  "scene": "good-night",
  "message": "Scene 'good-night' triggered (stub)"
}
```

### Scene Alias

```
POST /scene/{scene_id}/run
Headers: x-api-key
```

### Toggle Device

```
POST /device/{device_id}/toggle
Headers: x-api-key

200 OK
{
  "ok": true,
  "device_id": "livingroom-lamp",
  "action": "toggle",
  "message": "Toggled (stub)"
}
```

### Set Device State

```
POST /device/{device_id}/set
Headers: x-api-key
Body JSON: { "brightness": 0-100, "temp": 2000-6500, "mode": "string" }

200 OK
{
  "ok": true,
  "device_id": "livingroom-lamp",
  "action": "set",
  "applied": { "brightness": 70, "temp": 3000, "mode": "reading" },
  "message": "State set (stub)"
}
```

### Error Shapes

```
401 Unauthorized → { "detail": "Missing x-api-key" }
403 Forbidden    → { "detail": "Invalid API key" }
404 Not Found    → { "detail": "Unknown scene" }
```

---

## Examples

### iOS Shortcut: **Good Night 🌙**

* **URL**: `https://api.yourdomain.com/quick/good-night`
* **Method**: `POST`
* **Header**: `x-api-key: <your-secret-key>`
* Optional: Show the JSON `message` in a notification.

### Curl

```bash
curl -s https://api.yourdomain.com/health | jq
curl -s -X POST https://api.yourdomain.com/quick/focus -H "x-api-key: $API_KEY" | jq
```

---

## Security Notes

* Treat `API_KEY` like a password; rotate periodically.
* Consider basic **rate limiting** at API Gateway (usage plans) or a lightweight middleware later.
* Prefer **HTTPS** with a custom domain (ACM cert) in production.

---

## Roadmap

* Hue/WiZ/LIFX LAN adapters (replace stubs)
* Home Assistant bridge (REST/WebSocket) for Alexa/Google/HomeKit/LG
* Scheduled routines and scenes
* Dashboard UI (React)
* CI/CD with GitHub Actions (lint, test, deploy)

---

## Contributing

PRs welcome! Please open an issue for feature requests or questions.

---

## License

MIT — see `LICENSE`.

---

## Contact

Built by **Gabrielle Masten** — [https://rockthislife.com](https://rockthislife.com)
