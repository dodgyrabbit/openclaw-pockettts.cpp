# PocketTTS.cpp in Docker for OpenClaw
## Overview

You want to add good **Text to speech** support to OpenClaw, but you don't want to pay for third party services? This is the place for you!

Since PocketTTS.cpp already exposes an OpenAI endpoint, this repo helps you easily set it up in a Docker container and walks you through the process of configuring it in OpenClaw. You do not need to install any other OpenClaw plugins, as the OpenAI TTS Plugin is available by default and we're going to emulate that endpoint.

* Run [PocketTTS.cpp](https://github.com/VolgaGerm/PocketTTS.cpp), a super fast Text To Speech engine, *locally* on system for your [OpenClaw](http://openclaw.ai/) deployment.
* **No GPU** required, CPU only - saves your expensive GPU and memory for your Agent.
* Runs in a **Docker container**.
* Exposes the **OpenAI compatible** [Create Speech](https://developers.openai.com/api/reference/resources/audio/subresources/speech/methods/create) endpoint. No **subscriptions or API keys** required.

## What's in the repo

This repository contains a standalone Dockerized build of [PocketTTS.cpp](https://github.com/VolgaGerm/PocketTTS.cpp).

It is a multi-stage image that:

- builds `PocketTTS.cpp` from source
- exports/converts PocketTTS models to ONNX
- downloads a default voice sample (`alba`)
- pre-warms voice cache (`voices/.cache`) during build
- runs the PocketTTS.cpp **HTTP server** at container startup

## Build the container
Before we can start the container and configure OpenClaw, we need to build the Docker container. This should take a few minutes. Note that it will download the required model, convert it to ONNX format and configure a default voice.
```bash
 docker build -t pockettts-cpp:local -f Dockerfile .
```

There are two scenarios for your next step and this depends on how you installed OpenClaw. If you're running it directly on the host machine, read the next section, otherwise skip to [OpenClaw in Docker](#openclaw-in-docker).

## OpenClaw on Host

### Start the container

If you have installed OpenClaw on your host machine (as opposed to in Docker), run the following command to start the container. In this mode, the container is bound to localhost on port 8711.

```bash
docker compose up -d
```

You should be able to browse to the health check endpoint: [http://127.0.0.1:8711/health](http://127.0.0.1:8711/health)

Verify the audio generation is working with the following command. It will write the audio to a file called `output.wav` in the current directory.

```shell
curl http://localhost:8711/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{
    "input": "Hello from the native OpenClaw test",
    "voice": "alba",
    "response_format": "wav"
  }' \
  --output output.wav
```

### Configure OpenClaw

```shell
openclaw config set --batch-json '[
    { "path": "messages.tts.provider", "value": "openai" },
    { "path": "messages.tts.auto", "value": "always" },
    { "path": "messages.tts.providers.openai.apiKey", "value": "ignored" },
    { "path": "messages.tts.providers.openai.baseUrl", "value": "http://localhost:8711/v1" },
    { "path": "messages.tts.providers.openai.model", "value": "ignored" },
    { "path": "messages.tts.providers.openai.voice", "value": "alba" },
    { "path": "messages.tts.providers.openai.responseFormat", "value": "wav" }
  ]'
```

Restart the gateway to apply it:

```shell
openclaw gateway restart
```

Skip to [Testing it in OpenClaw](#test-it-in-openclaw).

## OpenClaw in Docker

### Start the container

If you have installed OpenClaw in a Docker container, run the following command to start the container. In this mode, the container is bound exposed to the OpenClaw Docker network on port 8000.

```bash
docker compose -f docker-compose.openclaw-network.yml up -d
```

This will create the container, alias it as `pockettts-cpp` and bind it to the default OpenClaw docker network `openclaw_default`. This is relevant to understand, since when we configure our TTS in OpenClaw, we address it by the service name `pockettts-cpp`.

### Configure OpenClaw

This section assumes you've installed [ClawDock](https://docs.openclaw.ai/install/clawdock#clawdock).
To configure OpenClaw to use the OpenAI TTS endpoint, use the following command:

```shell
clawdock-cli config set --batch-json '[
    { "path": "messages.tts.provider", "value": "openai" },
    { "path": "messages.tts.auto", "value": "always" },
    { "path": "messages.tts.providers.openai.apiKey", "value": "ignored" },
    { "path": "messages.tts.providers.openai.baseUrl", "value": "http://pockettts-cpp:8000/v1" },
    { "path": "messages.tts.providers.openai.model", "value": "ignored" },
    { "path": "messages.tts.providers.openai.voice", "value": "alba" },
    { "path": "messages.tts.providers.openai.responseFormat", "value": "wav" }
  ]'
```

Restart the gateway to apply it:

```shell
clawdock-cli gateway restart
```

## Test it in OpenClaw

* In your Agent chat session, use the command `/tts` to view available commands
* While it should already be set, you can use `/tts provider openai` to configure it to use the OpenAI provider we added
* `/tts audio This is a test` should respond with a wav file corresponding to "This is a test"
* `/tts on` will configure it to always respond with an audio version of the Agent's response

## API exposed by PocketTTS.cpp

- `POST /v1/audio/speech` (OpenAI-compatible)
- `POST /tts` (JSON streaming endpoint)
- `GET /health`


## Notes

- Voice/cache persistence is via named volume mounted at `/app/voices`.
- Image defaults to HTTP server mode (`--server --port 8000`).
