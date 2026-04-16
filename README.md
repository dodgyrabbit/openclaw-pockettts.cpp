# PocketTTS.cpp Docker Sidecar for OpenClaw

This repository contains a standalone Dockerized build of [PocketTTS.cpp](https://github.com/VolgaGerm/PocketTTS.cpp).

It is a multi-stage image that:

- builds `PocketTTS.cpp` from source
- exports/converts PocketTTS models to ONNX
- downloads a default voice sample (`alba`)
- pre-warms voice cache (`voices/.cache`) during build
- runs the PocketTTS.cpp **HTTP server** at container startup

## Quick start

```bash
cp .env.example .env   # optional
docker compose up -d --build
```

Default host endpoint:

- `http://127.0.0.1:8711`

Health check:

```bash
curl -fsS http://127.0.0.1:8711/health
```

## API exposed by PocketTTS.cpp

- `POST /v1/audio/speech` (OpenAI-compatible)
- `POST /tts` (JSON streaming endpoint)
- `GET /health`

## Docker-to-Docker mode (OpenClaw network)

```bash
OPENCLAW_NETWORK=openclaw_default docker compose \
  -f docker-compose.openclaw-network.yml \
  up -d --build
```

Then use:

- `http://pockettts:8000`

## Notes

- Voice/cache persistence is via named volume mounted at `/app/voices`.
- Image defaults to HTTP server mode (`--server --port 8000`).
