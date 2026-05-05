# OpenClaw CLI

tank-os provides a host-side `openclaw` wrapper at `/usr/local/bin/openclaw`.
It delegates into the running OpenClaw container, so users do not need to install
a separate Node.js/OpenClaw CLI on the host and do not need to open an
interactive shell inside the container for normal operations.

For the default instance:

```bash
openclaw gateway status --deep
openclaw doctor
openclaw dashboard --no-open
openclaw devices list
openclaw devices approve <request-id>
```

## Dashboard URL

To print the OpenClaw dashboard URL from the VM:

```bash
openclaw dashboard --no-open
```

If `gateway.auth.token` is configured as a plain token, the URL includes it as a
fragment, for example `http://127.0.0.1:18789/#token=...`. If the token is
SecretRef-managed, OpenClaw intentionally prints a non-tokenized URL and asks you
to use the external token source instead.

The wrapper targets the `openclaw` container by default. To target another
container, either use `--container`:

```bash
openclaw --container openclaw-research doctor
```

or set `OPENCLAW_CONTAINER`:

```bash
export OPENCLAW_CONTAINER=openclaw-research
openclaw doctor
```

For low-level debugging, use Podman directly:

```bash
podman exec -it openclaw sh
podman logs -f openclaw
```

The Podman shell path is an escape hatch, not the main UX.

## Single Instance And Multiple Instances

The bootc image ships one default rootless Quadlet:

```text
/etc/containers/systemd/users/1000/openclaw.container
```

That keeps first boot predictable and gives the machine one obvious gateway,
state directory, and service name:

```text
~/.openclaw
openclaw.service
openclaw
```

Multiple instances are still possible, but they should be explicit. A local
multi-instance shape should follow the installer model:

- unique container names, for example `openclaw-<prefix>-<name>`
- unique user services, for example `openclaw-<prefix>-<name>.service`
- unique data directories or volumes
- unique host ports for gateway and bridge
- per-instance env/config describing image, ports, secrets, and token

For tank-os, the likely next step is to add an installer-style `tank-openclaw`
instance manager that writes per-instance Quadlets and SecretRef config. Until
then, the image intentionally starts one default gateway and the wrapper can
target additional manually-created containers by name.
