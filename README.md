# Gong Chat Backend

The Gong Chat backend is a small Flask service that helps a static GitHub Pages frontend recommend a Gong cha drink. It keeps the OpenAI API key on the server, supplies the model with a categorized starter catalog, validates incoming chat data, and returns assistant replies as JSON.

The catalog is based on the [official Gong cha product guide](https://www.gong-cha.com/our-products/). Product names and availability can vary by country and individual shop.

## API

### `GET /health`

A lightweight deployment check. It does not call OpenAI.

Example response:

```json
{
  "status": "ok",
  "service": "gong-cha-chat"
}
```

### `POST /chat`

Generates one conversational drink-guide response.

Request body:

```json
{
  "message": "I want something creamy and not too sweet",
  "history": [
    {"role": "user", "content": "I like floral tea"},
    {"role": "assistant", "content": "Would you prefer something creamy or refreshing?"}
  ]
}
```

- `message` is required and must be a non-empty string of no more than 800 characters.
- `history` is optional. It must be an array of `user` and `assistant` messages. The backend keeps at most the latest 12 entries and truncates each content value to 800 characters.

Successful response:

```json
{
  "reply": "I recommend **Green Tea with Milk Foam** because..."
}
```

The assistant is instructed to ask short preference questions, recommend an exact catalog item, format the recommended drink name in Markdown bold, and treat toppings as customizations. The frontend renders the bold drink name as styled text.

Possible error responses include:

- `400` for invalid JSON, a missing/empty message, or an oversized message.
- `429` when the per-IP in-memory request limit is reached.
- `502` when the OpenAI completion cannot be produced.

## Frontend communication

The static frontend is hosted separately on GitHub Pages in the `Gong chat` folder of the website repository. Its `chat.html` page:

1. Adds the user’s message to the visible conversation.
2. Sends a `POST /chat` request to the configured Render URL with the current message and the latest conversation history.
3. Receives the JSON `{ "reply": "..." }` response.
4. Adds the assistant reply to the chat and stores it in the next request’s history.
5. Displays a generic retry message when the request fails.

The frontend does not call `/health` during normal conversation. `/health` is available for Render deployment checks and manual smoke tests. Because the frontend sends JSON, the Render service must allow the GitHub Pages origin through `ALLOWED_ORIGINS`.

## Local setup

Requirements: Python 3.10 or newer and an OpenAI API key.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
export OPENAI_API_KEY="your-openai-api-key"
export ALLOWED_ORIGINS="http://localhost:8000"
export OPENAI_MODEL="gpt-4o-mini"
python app.py
```

The API runs at `http://localhost:5000`. Check it with:

```bash
curl http://localhost:5000/health
```

For a local browser test, serve the frontend repository from a separate terminal:

```bash
python3 -m http.server 8000
```

Then configure the frontend’s `BACKEND_URL` as `http://localhost:5000` and open its `chat.html` page through the local HTTP server. Do not open the HTML directly as a `file://` URL because browser CORS behavior differs.

A local `private.txt` file is also supported by the current application as a development fallback, but it is ignored by Git. Environment variables are the recommended approach.

## Render deployment

This repository includes a root-level `render.yaml`. When creating the Render web service from this repository, leave **Root Directory** blank because `app.py` and `requirements.txt` are at the repository root.

Use:

```text
Build Command: pip install -r requirements.txt
Start Command: gunicorn --bind 0.0.0.0:$PORT app:app
```

Set these Render environment variables:

```text
OPENAI_API_KEY=<your OpenAI key>
ALLOWED_ORIGINS=https://izzyh-gif.github.io
OPENAI_MODEL=gpt-4o-mini
```

Use a comma-separated value if you need multiple exact origins, for example:

```text
https://izzyh-gif.github.io,http://localhost:8000
```

After deployment, verify `https://<your-render-service>.onrender.com/health` returns the health JSON. Update the GitHub Pages frontend’s `BACKEND_URL` to that service URL if it changes.

## Secrets and authentication

The OpenAI API key is read by the backend from the `OPENAI_API_KEY` environment variable. It is never included in the frontend response or sent to the browser. Render stores the production value in its environment settings; local development should use an environment variable or an ignored `private.txt` file.

The repository `.gitignore` excludes API-key files, `.env` files, Python caches, and editor artifacts. Never commit a real key. If a key is exposed, revoke it and create a replacement in OpenAI.

This service does not currently implement user authentication. The `/chat` endpoint is intentionally public for the static site, with CORS, input limits, generic errors, and a small in-memory rate limit as baseline protections. CORS is not authentication; add a stronger abuse-control or authentication layer before using it for a high-traffic or sensitive deployment.
