[English](README.md) | [简体中文](README-zh.md) | [繁體中文](README-zh-Hant.md) | [Русский](README-ru.md)

# Self-Hosted AI Stack

[![Powered by Docker Compose](docs/images/powered-by-docker-compose.svg)](https://docs.docker.com/compose/) &nbsp;[![Docker Pulls](https://raw.githubusercontent.com/hwdsl2/badges/main/img/docker-pulls-ai-stack.svg)](https://hub.docker.com/u/hwdsl2) &nbsp;[![License: MIT](docs/images/license.svg)](https://opensource.org/licenses/MIT)

<p align="center">
  <img src="docs/images/self-hosted-ai-stack-overview.png"
       alt="Self-Hosted AI Stack: deploy a complete self-hosted AI stack with one command"
       width="100%">
</p>

Run local AI models, chat with your documents, transcribe audio, and connect MCP tools with Docker Compose. Optional services add document parsing, text-to-speech, and real-time transcription.

**Features:**

- **Zero-config:** all services auto-configure on first start.
- **Secure by default:** AnythingLLM password protection is enabled, and bundled API services auto-generate keys.
- **HTTPS-ready:** optional Caddy overlay provides automatic TLS and binds direct HTTP ports to localhost.
- **Local by default:** runs locally, with optional external provider support via GatewayCrate.
- **AI agent ready:** connect [goose](https://github.com/aaif-goose/goose) on your workstation to GatewayCrate with the [setup guide](https://selfhostedaistack.com/goose).
- **Flexible configuration:** customize models, ports, providers, and API keys with simple env files.
- **[Lightweight stacks](#lightweight-stacks):** for lower memory requirements (as low as ~4.5 GB).
- **GPU acceleration:** via NVIDIA CUDA.
- **Multi-architecture support:** `linux/amd64`, `linux/arm64`.

> 📘 [The Self-Hosted AI Builder’s Guide](https://books2read.com/aiguide?store=amazon) is the hands-on companion to this stack, covering deployment, security, backups, and upgrades.

## Included services

| Service | Role | Default port |
|---|---|---|
| **[InferCrate (Ollama LLM)](https://github.com/hwdsl2/infercrate)** | Runs local LLM models (llama3, qwen, mistral, etc.) | `11434` |
| **[AnythingLLM](https://github.com/mintplex-labs/anything-llm)** | Web-based chat UI — password-protected by default | `3001` |
| **[GatewayCrate (LiteLLM)](https://github.com/hwdsl2/gatewaycrate)** | AI gateway with Admin UI — routes requests to InferCrate and 100+ providers | `4000` |
| **[EmbedCrate (Hugging Face TEI)](https://github.com/hwdsl2/embedcrate)** | Converts text to vectors for semantic search and RAG | `8000` |
| **[ScribeCrate (Whisper STT)](https://github.com/hwdsl2/scribecrate)** | Transcribes spoken audio to text | `9000` |
| **[ScribeCrate Live (real-time STT)](https://github.com/hwdsl2/scribecrate-live)** | Real-time speech-to-text transcription over WebSocket | `9090` |
| **[SpeakCrate (Kokoro TTS)](https://github.com/hwdsl2/speakcrate)** | Converts text to natural-sounding speech | `8880` |
| **[UplinkCrate (MCPHub)](https://github.com/hwdsl2/uplinkcrate)** | Provides MCP tools (filesystem, fetch, GitHub, search, databases) to AI clients | `3000` |
| **[ParseCrate (Docling)](https://github.com/hwdsl2/parsecrate)** | Converts documents (PDF, DOCX, etc.) to structured text/Markdown | `5001` |

## Quick start

**Requirements:**

- A Linux server (local or cloud) with Docker installed
- At least 8 GB of RAM (with small models). For larger LLM models (8B+), 16 GB or more is recommended.
- You can comment out services you don't need to reduce memory usage.

**Start the full stack:**

```bash
# Clone the repository to get the compose files
git clone https://github.com/hwdsl2/self-hosted-ai-stack
cd self-hosted-ai-stack
docker compose up -d
```

> **PostgreSQL credentials:** Fresh installs and existing default installs are handled automatically. If you previously set a custom database password, see [PostgreSQL credentials](#postgresql-credentials) before starting.

**Pull a model** (required before making LLM requests):

```bash
docker exec ollama ollama_manage --pull llama3.2:3b
```

Run the health check to verify all services are working:

```bash
./stack-check.sh
```

> **Tip:** On first start, services may take a few minutes to initialize. If any checks fail, wait and run `./stack-check.sh` again. Use `docker compose logs` to check progress.

For detailed troubleshooting, see the [Troubleshooting](docs/troubleshooting.md) guide.

**Get the GatewayCrate master key** (used to log into the Admin UI and for LLM requests):

```bash
docker exec litellm litellm_manage --showkey
```

<details>
<summary>Show core API keys (InferCrate, GatewayCrate, UplinkCrate)</summary>

```bash
docker exec ollama ollama_manage --showkey
docker exec litellm litellm_manage --showkey
docker exec mcp mcp_manage --showkey
```

</details>

**Access AnythingLLM (Chat UI):**

AnythingLLM is pre-configured to connect to your local LLM via GatewayCrate. On first start, it may take a few minutes to become available (check progress with `docker logs anythingllm`).

**Password-protected by default.** A random admin password is auto-generated on first start, printed once to `docker logs anythingllm`, and saved to `/app/server/storage/.initial_admin_password` inside the `anythingllm-data` volume. The seeded password persists across container upgrades. Change it any time from **Settings → Security**; after you do, `.initial_admin_password` may no longer match the current login password.

Retrieve the auto-generated password:

```bash
# At any time from the data volume:
docker exec anythingllm cat /app/server/storage/.initial_admin_password

# Or from the live logs (only shown on first start):
docker compose logs anythingllm | grep -A4 "FIRST RUN"
```

Open `http://<server-ip>:3001` in your browser and log in with the password above.

> [!NOTE]
> When exposing AnythingLLM beyond `localhost` or a trusted LAN, use the included Caddy HTTPS overlay so the password is encrypted in transit and direct HTTP ports are bound to localhost. See [Internet-facing deployments](#internet-facing-deployments) below.

**Access GatewayCrate's LiteLLM Admin UI:**

Open `http://<server-ip>:4000/ui` in your browser. Log in with username `admin` and your GatewayCrate master key as the password. The UI provides virtual key management, spend tracking, and model configuration.

> **Tip:** In the Admin UI, click **Playground** in the left menu. Select a local model (e.g., `ollama-chat/llama3.2:3b`) from the dropdown and start chatting — a quick way to verify your local LLM is working end-to-end.

**Use the stack with goose:**

Install goose on your workstation and connect it to the stack's GatewayCrate endpoint for supervised agent and coding tasks. The [goose setup guide](https://selfhostedaistack.com/goose) covers native installation, a restricted GatewayCrate key, local-model expectations, permissions, testing, and optional UplinkCrate access.

**Stop the stack:**

```bash
# Stop and remove all containers (data is preserved in Docker volumes)
docker compose down
```

## Community

- 📬 [Get project updates and free deployment guides](https://selfhostedstack.beehiiv.com/subscribe?utm_campaign=ai) (1–2 emails/month)
- 💬 Join the [r/selfhostedstack](https://www.reddit.com/r/selfhostedstack/) community for discussions and showcases
- ⭐ Star the repository if you find it useful — it helps others discover it

Self-Hosted AI Stack is maintained by the author of [Setup IPsec VPN](https://github.com/hwdsl2/setup-ipsec-vpn) (28k+ stars).

## GPU acceleration (NVIDIA CUDA)

For NVIDIA GPU acceleration, use the CUDA compose file:

```bash
docker compose -f docker-compose.cuda.yml up -d
```

> **Tip:** To avoid adding `-f docker-compose.cuda.yml` to every subsequent `docker compose` command (`down`, `pull`, `logs`, etc.), set it once for your shell session:
>
> ```bash
> export COMPOSE_FILE=docker-compose.cuda.yml
> ```
>
> Then run plain `docker compose` commands as usual. To make it persistent, add `COMPOSE_FILE=docker-compose.cuda.yml` to a `.env` file in this directory. Run `unset COMPOSE_FILE` to switch back to the CPU configuration.

**Requirements:** NVIDIA GPU, [NVIDIA driver](https://www.nvidia.com/en-us/drivers/) 575.57.08+ (Linux) or 576.57+ (Windows), and the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) installed on the host. CUDA images are `linux/amd64` only.

> [!IMPORTANT]
> The Compose `deploy:` GPU block is ignored by Podman. Use CDI instead — see [Using Podman](#using-podman).

## Lightweight stacks

Don't need the full stack? Use a pre-configured subset from the `stacks/` folder:

> [!IMPORTANT]
> The lightweight stacks share default container names, ports, and Docker volume names. Run one stack variant at a time with the default compose files; stop the current variant before switching to another. To combine capabilities, use the full stack or customize Compose project names, container names, ports, and volumes.

| Stack | Services | Memory | Use case |
|---|---|---|---|
| **[chat-ui](stacks/chat-ui/)** | InferCrate + GatewayCrate + AnythingLLM | ~5 GB | Web-based ChatGPT-like chat interface |
| **[voice-pipeline](stacks/voice-pipeline/)** | ScribeCrate + InferCrate + GatewayCrate + SpeakCrate | ~6 GB | Speech-to-text → LLM → text-to-speech |
| **[voice-chat](stacks/voice-chat/)** | ScribeCrate + InferCrate + GatewayCrate + SpeakCrate + AnythingLLM | ~6.5 GB | Chat UI with voice input/output |
| **[rag-pipeline](stacks/rag-pipeline/)** | InferCrate + GatewayCrate + EmbedCrate | ~5 GB | Semantic search + LLM Q&A |
| **[rag-pipeline-full](stacks/rag-pipeline-full/)** | InferCrate + GatewayCrate + EmbedCrate + ParseCrate | ~6 GB | Document parsing + semantic search + LLM Q&A |
| **[code-assistant](stacks/code-assistant/)** | InferCrate + GatewayCrate + UplinkCrate + EmbedCrate | ~5 GB | AI coding with tools + semantic code search |
| **[ai-tools](stacks/ai-tools/)** | InferCrate + GatewayCrate + UplinkCrate | ~5 GB | AI coding assistant with tool access |
| **[chat-only](stacks/chat-only/)** | InferCrate + GatewayCrate | ~4.5 GB | Minimal local ChatGPT replacement |

```bash
git clone https://github.com/hwdsl2/self-hosted-ai-stack
cd self-hosted-ai-stack/stacks/chat-ui  # or voice-pipeline, voice-chat, rag-pipeline, rag-pipeline-full, code-assistant, ai-tools, chat-only
docker compose up -d
```

## Architecture

```mermaid
graph LR
    A["🎤 Audio input"] -->|transcribe| W["ScribeCrate<br/>(speech-to-text)"]
    D["📄 Documents"] -->|parse| DC["ParseCrate<br/>(document → text)"]
    DC -->|embed| E["EmbedCrate<br/>(text → vectors)"]
    E -->|store| VDB["pgvector<br/>(in shared Postgres)"]
    W -->|query| E
    VDB -->|context| L["GatewayCrate<br/>(AI gateway)"]
    W -->|text| L
    L -->|routes to| O["InferCrate<br/>(local LLM)"]
    L -->|response| T["SpeakCrate TTS<br/>(text-to-speech)"]
    T --> B["🔊 Audio output"]
    C["🤖 AI client<br/>(goose, Cline, Claude, etc.)"] -->|MCP tools| M["UplinkCrate<br/>(MCP endpoint)"]
    C -->|chat| L
    L -->|MCP protocol| M
    U["👤 User"] -->|chat| AN["AnythingLLM<br/>(chat UI)"]
    AN -->|LLM requests| L
    AN -->|MCP tools| M
    U -->|use| C
    U -->|speak| A
    U -->|upload| D
```

**Notes:**

- InferCrate's port (`11434`) and UplinkCrate's port (`3000`) are internal to the Docker network and not exposed to the host by default. Access your LLM through GatewayCrate on port `4000`.
- SpeakCrate (TTS), ParseCrate (document parsing), and ScribeCrate Live (real-time STT) are disabled by default to reduce memory usage. Uncomment these services in `docker-compose.yml` to enable them.

## Running without Docker Compose

If you prefer using `docker run` commands directly, first create a shared network so services can communicate:

```bash
docker network create ai-stack
```

Then generate a PostgreSQL password and start each service on the shared network:

> **Note:** With manual `docker run`, start PostgreSQL first, then the model and tool services included in your stack. Wait for dependencies to be ready before starting GatewayCrate, then start AnythingLLM if included. The examples below generate one PostgreSQL password variable and reuse it for Postgres and GatewayCrate.

```bash
LITELLM_POSTGRES_PASSWORD=$(LC_ALL=C tr -dc 'A-Za-z0-9' </dev/urandom | head -c 32)

# PostgreSQL with pgvector (required by GatewayCrate; pgvector enables vector storage for RAG)
docker run -d --name litellm-db --restart always \
    --network ai-stack \
    -e POSTGRES_USER=litellm \
    -e POSTGRES_PASSWORD="$LITELLM_POSTGRES_PASSWORD" \
    -e POSTGRES_DB=litellm \
    -v litellm-db:/var/lib/postgresql \
    pgvector/pgvector:pg18-trixie

# InferCrate (LLM)
docker run -d --name ollama --restart always \
    --network ai-stack \
    -v ollama-data:/var/lib/ollama \
    -v ollama-shared:/var/lib/ollama-shared \
    hwdsl2/ollama-server

# UplinkCrate
docker run -d --name mcp --restart always \
    --network ai-stack \
    -v mcp-data:/var/lib/mcp \
    -v mcp-shared:/var/lib/mcp-shared \
    hwdsl2/mcp-gateway

# GatewayCrate (AI gateway)
docker run -d --name litellm --restart always \
    --network ai-stack \
    -p 4000:4000 \
    -e LITELLM_OLLAMA_BASE_URL=http://ollama:11434 \
    -e LITELLM_MCP_URL=http://mcp:3000/mcp \
    -e LITELLM_DATABASE_URL="postgresql://litellm:${LITELLM_POSTGRES_PASSWORD}@litellm-db:5432/litellm" \
    -v litellm-data:/etc/litellm \
    -v ollama-shared:/var/lib/ollama-shared:ro \
    -v mcp-shared:/var/lib/mcp-shared:ro \
    -v litellm-shared:/var/lib/litellm-shared \
    hwdsl2/litellm-server

# EmbedCrate
docker run -d --name embeddings --restart always \
    --network ai-stack \
    -p 127.0.0.1:8000:8000 \
    -v embeddings-data:/var/lib/embeddings \
    hwdsl2/embeddings-server

# ScribeCrate (Whisper STT)
docker run -d --name whisper --restart always \
    --network ai-stack \
    -p 127.0.0.1:9000:9000 \
    -v whisper-data:/var/lib/whisper \
    hwdsl2/whisper-server

# ScribeCrate Live (real-time STT)
docker run -d --name whisper-live --restart always \
    --network ai-stack \
    -p 127.0.0.1:9090:9090 \
    -v whisper-live-data:/var/lib/whisper-live \
    hwdsl2/whisper-live-server

# AnythingLLM (chat UI)
docker run -d --name anythingllm --restart always \
    --network ai-stack \
    -p 3001:3001 \
    -e STORAGE_DIR=/app/server/storage \
    -e LLM_PROVIDER=generic-openai \
    -e GENERIC_OPEN_AI_BASE_PATH=http://litellm:4000/v1 \
    -e GENERIC_OPEN_AI_MODEL_PREF=ollama/llama3.2:3b \
    -e GENERIC_OPEN_AI_MODEL_TOKEN_LIMIT=131072 \
    -e ANYTHINGLLM_DEFAULT_CHAT_MODE=chat \
    -e EMBEDDING_ENGINE=native \
    -e DISABLE_TELEMETRY=true \
    -v anythingllm-data:/app/server/storage \
    -v litellm-shared:/var/lib/litellm-shared:ro \
    -v "$(pwd)/chat-ui-bootstrap.sh:/usr/local/bin/chat-ui-bootstrap.sh:ro" \
    --entrypoint /bin/bash \
    mintplexlabs/anythingllm:1.16.2 \
    /usr/local/bin/chat-ui-bootstrap.sh

# SpeakCrate (TTS)
docker run -d --name kokoro --restart always \
    --network ai-stack \
    -p 127.0.0.1:8880:8880 \
    -v kokoro-data:/var/lib/kokoro \
    hwdsl2/kokoro-server

# ParseCrate (document parsing)
docker run -d --name docling --restart always \
    --network ai-stack \
    -p 127.0.0.1:5001:5001 \
    -v docling-data:/var/lib/docling \
    hwdsl2/docling-server
```

**Note:** The shared network allows services to reach each other by container name (e.g., GatewayCrate connects to InferCrate via `http://ollama:11434`). You can start only the services you need — they don't all have to run together.

**Pull a model** (required before making LLM requests):

```bash
docker exec ollama ollama_manage --pull llama3.2:3b
```

## Using Podman

The stack runs under [Podman](https://podman.io/) on a best-effort basis. The CPU compose files work as-is; GPU acceleration and SELinux-enabled hosts need a couple of extra steps described below. Podman **4.1+** is recommended.

**1. Install the Docker CLI shim.** So that the `docker` commands in this README and the `stack-check.sh` health check work unchanged, install the `podman-docker` package (provides a `docker` → `podman` wrapper):

```bash
# Fedora / RHEL / CentOS Stream
sudo dnf install -y podman-docker

# Debian / Ubuntu
sudo apt-get install -y podman-docker
```

> [!NOTE]
> A shell `alias docker=podman` is **not** sufficient — aliases are not seen by scripts such as `stack-check.sh`. Use the `podman-docker` package (or a `docker` → `podman` symlink in your `PATH`) instead. Alternatively, `stack-check.sh` auto-detects Podman; you can also force it with `CONTAINER_ENGINE=podman ./stack-check.sh`.

**2. Install a Compose provider.** `podman compose` delegates to an external provider. Install either `podman-compose` or `docker-compose`:

```bash
# Fedora / RHEL / CentOS Stream
sudo dnf install -y podman-compose

# Debian / Ubuntu
sudo apt-get install -y podman-compose
```

**3. Start the stack.** With the shim installed, every command in this README works unchanged. Without it, substitute `podman` for `docker`:

```bash
git clone https://github.com/hwdsl2/self-hosted-ai-stack
cd self-hosted-ai-stack
podman compose up -d
```

Run the health check (auto-detects the engine):

```bash
./stack-check.sh
```

**GPU acceleration (CDI).** Podman does not read the Compose `deploy.resources` GPU block. Instead, use the [Container Device Interface (CDI)](https://github.com/cncf-tags/container-device-interface). After installing the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html), generate a CDI spec:

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
```

Then expose the GPU to the relevant services. For `podman compose`, replace the `deploy:` block in `docker-compose.cuda.yml` with a `devices:` entry for the `ollama` (and `whisper`) services:

```yaml
    devices:
      - nvidia.com/gpu=all
```

For a plain `podman run` command, add `--device nvidia.com/gpu=all`.

**SELinux.** On SELinux-enabled hosts (Fedora, RHEL, CentOS Stream), bind-mounted files need a relabel suffix, or the container will be denied access. Add `:z` (shared) to the `chat-ui-bootstrap.sh` bind mount:

- In `docker-compose.yml`: change `./chat-ui-bootstrap.sh:/usr/local/bin/chat-ui-bootstrap.sh:ro` to `./chat-ui-bootstrap.sh:/usr/local/bin/chat-ui-bootstrap.sh:ro,z`
- In the `podman run` command above: change `"$(pwd)/chat-ui-bootstrap.sh:/usr/local/bin/chat-ui-bootstrap.sh:ro"` to `"$(pwd)/chat-ui-bootstrap.sh:/usr/local/bin/chat-ui-bootstrap.sh:ro,z"`

Named volumes do not need relabeling.

**Next steps:** Pull a model and access the services — follow the instructions in [Quick start](#quick-start) starting from "Pull a model." With the `podman-docker` shim installed, all commands work unchanged.

## Connect UplinkCrate to GatewayCrate

GatewayCrate and UplinkCrate are **automatically wired** when using the compose files in this repository — no manual key setup is needed.

API keys are shared automatically between services via Docker shared volumes:

- InferCrate generates an API key on first start and copies it to a shared volume
- UplinkCrate does the same
- GatewayCrate reads both keys from the shared volumes on startup

The `LITELLM_MCP_URL=http://mcp:3000/mcp` and `LITELLM_OLLAMA_BASE_URL=http://ollama:11434` environment variables are pre-configured in the compose files, so all services are connected automatically with a single `docker compose up -d`.

Once connected, AI clients that call GatewayCrate can use MCP tools (filesystem, fetch, GitHub, etc.) directly through the GatewayCrate proxy.

## Voice pipeline example

Transcribe a spoken question, get a local LLM response via InferCrate, and convert it to speech:

**Note:** SpeakCrate (TTS) is disabled by default. To use this example, first uncomment the `kokoro` service in `docker-compose.yml`, then run `docker compose up -d`.

**Tip:** Need a sample audio file? Downlo