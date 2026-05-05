# Model Providers And Secrets

tank-os keeps model provider keys out of the image and out of `~/.openclaw/openclaw.json`.
Users add keys as rootless Podman secrets owned by the `openclaw` user, then run
`tank-openclaw-secrets` to sync Quadlet secret mounts and OpenClaw SecretRefs.

## Built-In Provider Keys

Create secrets after the machine boots, before starting or restarting the OpenClaw service:

```bash
sudo -iu openclaw
printf '%s' "$ANTHROPIC_API_KEY" | podman secret create anthropic_api_key -
printf '%s' "$OPENAI_API_KEY" | podman secret create openai_api_key -
printf '%s' "$GEMINI_API_KEY" | podman secret create gemini_api_key -
printf '%s' "$OPENROUTER_API_KEY" | podman secret create openrouter_api_key -
tank-openclaw-secrets
systemctl --user restart openclaw.service
```

Supported OpenClaw secret names:

| Podman secret | Container env | Notes |
| --- | --- | --- |
| `anthropic_api_key` | `ANTHROPIC_API_KEY` | Built-in `anthropic/*` models use env auth directly; the helper does not write a provider override. |
| `openai_api_key` | `OPENAI_API_KEY` | Built-in `openai/*` models use env auth directly; the helper does not write a provider override. |
| `gemini_api_key` | `GEMINI_API_KEY` | Adds `models.providers.google` with a SecretRef. |
| `google_api_key` | `GOOGLE_API_KEY` | Alternate Google env name; used only if `gemini_api_key` is absent. |
| `openrouter_api_key` | `OPENROUTER_API_KEY` | Adds `models.providers.openrouter` with a SecretRef. |
| `model_endpoint_api_key` | `MODEL_ENDPOINT_API_KEY` | Key only; custom endpoint metadata is still manual. |
| `telegram_bot_token` | `TELEGRAM_BOT_TOKEN` | Adds `channels.telegram.botToken` with a SecretRef. |

The helper writes rootless Quadlet drop-ins under
`~/.config/containers/systemd/openclaw.container.d/` and updates
`~/.openclaw/openclaw.json` with references like:

```json
{
  "secrets": {
    "providers": {
      "default": { "source": "env" }
    }
  },
  "models": {
    "providers": {
      "google": {
        "baseUrl": "https://generativelanguage.googleapis.com/v1beta",
        "api": "google-generative-ai",
        "apiKey": { "source": "env", "provider": "default", "id": "GEMINI_API_KEY" },
        "models": [{ "id": "gemini-3.1-pro-preview", "name": "gemini-3.1-pro-preview" }]
      }
    }
  }
}
```

This mirrors the installer model: Podman secrets supply environment variables,
and OpenClaw config references those env values through SecretRefs when explicit
provider metadata is needed. For built-in Anthropic and OpenAI models, the env
vars are enough, so `tank-openclaw-secrets` leaves the built-in provider catalog
in place.

## Custom Providers

A Podman secret by itself is not enough to create an arbitrary provider. OpenClaw
also needs the provider id, API adapter, base URL, and at least one model id.

Use a normal Podman secret for the key:

```bash
sudo -iu openclaw
printf '%s' "$MOONSHOT_API_KEY" | podman secret create moonshot_api_key -
mkdir -p ~/.config/containers/systemd/openclaw.container.d
cat > ~/.config/containers/systemd/openclaw.container.d/20-moonshot.conf <<'EOF'
[Container]
Secret=moonshot_api_key,type=env,target=MOONSHOT_API_KEY
EOF
systemctl --user daemon-reload
```

Then add the non-secret provider metadata to `~/.openclaw/openclaw.json`:

```json
{
  "secrets": {
    "providers": {
      "default": { "source": "env" }
    }
  },
  "models": {
    "providers": {
      "moonshot": {
        "baseUrl": "https://api.moonshot.ai/v1",
        "api": "openai-completions",
        "apiKey": { "source": "env", "provider": "default", "id": "MOONSHOT_API_KEY" },
        "models": [{ "id": "kimi-k2.5", "name": "Kimi K2.5" }]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": { "primary": "moonshot/kimi-k2.5" }
    }
  }
}
```

Restart after changing secrets or provider config:

```bash
systemctl --user restart openclaw.service
```

For now, `tank-openclaw-secrets` intentionally auto-configures only the known
provider mappings above. Custom providers remain explicit so the helper does not
guess an unsafe base URL or model catalog from a secret name.
