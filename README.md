# Halfmoon Helm Chart

Helm chart for deploying [Halfmoon](https://github.com/halfmoon-labs/halfmoon) — an ultra-lightweight personal AI assistant written in Go.

## Prerequisites

- Kubernetes 1.26+
- Helm 3.10+
- At least one LLM API key (Anthropic, OpenAI, etc.)

## Quick Start

```bash
helm repo add halfmoon https://halfmoon-labs.github.io/halfmoon-helm
helm repo update

helm install halfmoon halfmoon/halfmoon \
  --set config.agents.defaults.model_name=claude-sonnet-4-20250514 \
  --set credentials.security.model_list.claude-sonnet-4-20250514\:0.api_keys[0]=sk-ant-your-key
```

## Configuration

### Config (`values.config`)

The entire `config` block in `values.yaml` is serialized as JSON into a ConfigMap and mounted as `config.json`. Any field supported by Halfmoon's [config.json](https://github.com/halfmoon-labs/halfmoon/blob/main/config/config.example.json) can be added here.

```yaml
config:
  agents:
    defaults:
      model_name: "claude-sonnet-4-20250514"   # Required
      max_tokens: 8192
      temperature: 0.7
    list:
      - id: main
        default: true
        subagents:
          allow_agents: ["researcher"]
      - id: researcher
        model:
          primary: "gpt-4o"
        skills: ["web_search"]
  gateway:
    host: "0.0.0.0"
    port: 18790
    log_level: "info"
  channels:
    telegram:
      enabled: true
  tools: {}
```

### Secrets (`values.credentials`)

API keys and channel tokens are stored in a Kubernetes Secret as `.security.yml`.

```yaml
credentials:
  security:
    model_list:
      claude-sonnet-4-20250514:0:
        api_keys:
          - "sk-ant-..."
    channels:
      telegram:
        token: "123456:ABC..."
```

To use an existing Secret instead:

```yaml
credentials:
  existingSecret: "my-halfmoon-secrets"
```

The existing Secret must contain a `.security.yml` key with valid YAML content.

Alternatively, you can skip `.security.yml` entirely and pass secrets as environment variables:

```yaml
credentials:
  security: {}

extraEnv:
  - name: HALFMOON_CHANNELS_TELEGRAM_TOKEN
    valueFrom:
      secretKeyRef:
        name: my-external-secret
        key: telegram-token
```

Halfmoon does not require `.security.yml` to exist — environment variables work independently.

### Workspace Identity Files (`values.workspace`)

Agent identity files (`AGENT.md`, `SOUL.md`, etc.) can be declared directly in `values.yaml`. They are rendered into a ConfigMap and written to the workspace on every pod start, overwriting any existing copies — Helm is the source of truth for identity.

```yaml
workspace:
  # Root workspace files — written to workspace/
  files:
    AGENT.md: |
      ---
      name: Halfmoon
      description: General-purpose AI assistant
      ---
      You are Halfmoon, a helpful AI assistant.
    SOUL.md: |
      Be concise, accurate, and helpful.

  # Per-agent identity files — written to workspace/agents/{id}/
  agents:
    researcher:
      AGENT.md: |
        ---
        name: Research Specialist
        description: Finds and analyzes information from the web
        ---
        You are a research specialist. Focus on accuracy and cite sources.
    coder:
      AGENT.md: |
        ---
        name: Code Engineer
        description: Writes and reviews code
        ---
        You are a code engineer. Write clean, tested code.
```

### Multi-Agent Setup

Halfmoon supports specialized sub-agents with their own identity. Define agents in `config.agents.list` and their identity files in `workspace.agents`:

```yaml
config:
  agents:
    list:
      - id: main
        default: true
        subagents:
          allow_agents: ["researcher"]
      - id: researcher
        model:
          primary: "gpt-4o"
        skills: ["web_search"]

workspace:
  agents:
    researcher:
      AGENT.md: |
        ---
        name: Research Specialist
        description: Finds and analyzes information from the web
        ---
        You are a research specialist. Focus on accuracy and cite sources.
```

### Persistence

Workspace state (sessions, memory, skills, agent identity files) is persisted via a PVC.

```yaml
persistence:
  enabled: true        # default
  size: 5Gi
  storageClass: ""     # uses cluster default
  existingClaim: ""    # use an existing PVC
```

### Ingress

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
  hosts:
    - host: halfmoon.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: halfmoon-tls
      hosts:
        - halfmoon.example.com
```

## Values Reference

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `image.repository` | string | `ghcr.io/halfmoon-labs/halfmoon` | Container image |
| `image.tag` | string | Chart appVersion | Image tag |
| `image.pullPolicy` | string | `IfNotPresent` | Pull policy |
| `config` | object | — | Halfmoon config.json (serialized as JSON) |
| `config.agents.defaults.model_name` | string | `""` | **Required.** Primary LLM model |
| `config.agents.defaults.max_tokens` | int | `8192` | Max output tokens |
| `config.agents.defaults.temperature` | float | `0.7` | LLM temperature |
| `config.gateway.host` | string | `0.0.0.0` | Gateway listen address |
| `config.gateway.port` | int | `18790` | Gateway port |
| `config.gateway.log_level` | string | `info` | Log level |
| `workspace.files` | object | `{}` | Root workspace identity files (filename → content) |
| `workspace.agents` | object | `{}` | Per-agent identity files (agentId → {filename → content}) |
| `credentials.existingSecret` | string | `""` | Use existing Secret |
| `credentials.security` | object | `{}` | `.security.yml` content |
| `extraEnv` | list | `[]` | Extra environment variables |
| `serviceAccount.create` | bool | `true` | Create ServiceAccount |
| `persistence.enabled` | bool | `true` | Enable PVC for workspace |
| `persistence.size` | string | `5Gi` | PVC size |
| `persistence.accessMode` | string | `ReadWriteOnce` | PVC access mode |
| `persistence.storageClass` | string | `""` | Storage class (empty = default) |
| `persistence.existingClaim` | string | `""` | Use existing PVC |
| `service.type` | string | `ClusterIP` | Service type |
| `service.port` | int | `18790` | Service port |
| `ingress.enabled` | bool | `false` | Enable Ingress |
| `resources` | object | `{}` | CPU/memory requests and limits |
| `securityContext.runAsUser` | int | `1000` | Pod user ID |

## Health Checks

The chart configures liveness, readiness, and startup probes against:

- **Liveness**: `GET /health` (10s initial, 30s interval)
- **Readiness**: `GET /ready` (5s initial, 10s interval)
- **Startup**: `GET /health` (2s initial, 5s interval, 30 retries)

## Notes

- The deployment uses `strategy: Recreate` — Halfmoon is single-instance and cannot scale horizontally
- Config, secret, and workspace identity changes trigger a pod restart via checksum annotations
- The init container runs `halfmoon onboard` first, then copies any workspace identity files from the ConfigMap (always overwrites — Helm owns identity)
- All containers run as UID 1000 (non-root) with no privilege escalation

## License

MIT
