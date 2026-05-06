# Installation Guide

This guide walks you through getting tank-os running from scratch: building the
bootc container image, building a bootable disk image, booting it as a VM, and
completing first-boot configuration.

## Prerequisites

| Requirement | Notes |
| --- | --- |
| [Podman](https://podman.io/) | v4+ recommended; available on Linux and macOS |
| [Podman Desktop](https://podman-desktop.io/) *(optional)* | Easiest path on macOS for building disk images and running VMs |
| [Podman Desktop BootC extension](https://github.com/podman-desktop/extension-bootc) *(optional)* | GUI disk image builder |
| SSH key pair | `~/.ssh/id_ed25519` / `~/.ssh/id_ed25519.pub`; generate with `ssh-keygen -t ed25519` if needed |
| 20 GB free disk space | For the output QCOW2 image and container layers |

## Step 1 — Clone The Repository

```bash
git clone https://github.com/LobsterTrap/tank-os.git
cd tank-os
```

## Step 2 — Build The Bootc Container Image

> **Skip this step** if you want to use the pre-built published image
> `quay.io/sallyom/tank-os:latest` (available for both `arm64` and `amd64`).

Build from the repo root. The `bootc` argument at the end is the build context directory.

**Apple Silicon (arm64):**

```bash
podman build \
  --platform linux/arm64 \
  -t localhost/tank-os:latest \
  -f bootc/Containerfile \
  bootc
```

**x86-64 (amd64):**

```bash
podman build \
  --platform linux/amd64 \
  -t localhost/tank-os:latest \
  -f bootc/Containerfile \
  bootc
```

For a pinned Fedora base, pass `--build-arg FEDORA_BOOTC_BASE=quay.io/fedora/fedora-bootc:<tag>`.

## Step 3 — Build A Disk Image

Choose **one** of the methods below.

### Option A — Podman Desktop BootC Extension (recommended on macOS)

1. Open Podman Desktop and install the BootC extension if you have not already.
2. Enter `localhost/tank-os:latest` (or `quay.io/sallyom/tank-os:latest`) as the bootc image.
3. Choose these settings:

   | Setting | Value |
   | --- | --- |
   | Disk image type | `qcow2` |
   | Target architecture | `arm64` / `aarch64` (Apple Silicon) or `amd64` |
   | Root filesystem | `xfs` |
   | Output folder | A dedicated writable directory, e.g. `~/git/out-tank-os` |
   | User | `openclaw` |
   | SSH public key | Paste your `~/.ssh/id_ed25519.pub` content |
   | Groups | `wheel` |
   | Password | leave empty |

4. Click **Build**. The output is `<output-folder>/qcow2/disk.qcow2`.

### Option B — bootc-image-builder (manual)

Create an output directory and a config file with your SSH public key:

```bash
mkdir -p out-tank-os

cat > out-tank-os/config.json <<'EOF'
{
  "customizations": {
    "user": [
      {
        "name": "openclaw",
        "key": "ssh-ed25519 PASTE_YOUR_SSH_PUBLIC_KEY_HERE tank-os",
        "groups": ["wheel"]
      }
    ]
  }
}
EOF
```

Replace `PASTE_YOUR_SSH_PUBLIC_KEY_HERE` with the content of `~/.ssh/id_ed25519.pub`.

Run `bootc-image-builder` (on macOS, use the rootful Podman connection):

```bash
podman --connection podman-machine-default-root run \
  --rm \
  --name tank-os-bootc-image-builder \
  --tty \
  --privileged \
  --security-opt label=type:unconfined_t \
  -v "$PWD/out-tank-os:/output/" \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  -v "$PWD/out-tank-os/config.json:/config.json:ro" \
  quay.io/centos-bootc/bootc-image-builder:latest \
  localhost/tank-os:latest \
  --output /output/ \
  --local \
  --progress verbose \
  --type qcow2 \
  --target-arch arm64 \
  --rootfs xfs
```

Use `--target-arch amd64` for x86-64. The output is `out-tank-os/qcow2/disk.qcow2`.

## Step 4 — (Optional) Resize The Disk Image

The default image is 10 GB. The OpenClaw container image is ~3.5 GB, so resizing
to 20 GB before first boot avoids space issues. XFS grows automatically on boot.

```bash
qemu-img resize disk.qcow2 20G
```

## Step 5 — Boot The VM

Choose the method that matches your environment.

### macOS — QEMU with user-mode networking (arm64)

```bash
qemu-system-aarch64 \
  -machine virt,highmem=on \
  -accel hvf \
  -cpu host \
  -smp 4 \
  -m 4096 \
  -drive file=disk.qcow2,format=qcow2,if=virtio \
  -drive if=pflash,format=raw,unit=0,file=$(brew --prefix)/share/qemu/edk2-aarch64-code.fd,readonly=on \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -nographic
```

> **Note:** The firmware path above uses `brew --prefix`, which is correct for a
> Homebrew QEMU install. If you installed QEMU another way, replace that path
> with the location of `edk2-aarch64-code.fd` on your system.

SSH is forwarded to `localhost:2222`. Skip to Step 6 and use port `2222`.

### macOS — Podman Desktop / macadam VM

Start the disk image from Podman Desktop. The forwarded SSH port is managed by
`gvproxy`. Find it with:

```bash
export PORT="$(
  ps aux |
    grep 'gvproxy' |
    grep 'bootc.*tank' |
    sed -nE 's/.*-ssh-port ([0-9]+).*/\1/p' |
    tail -1
)"
echo "$PORT"
```

Use `$PORT` wherever `:2222` appears in the rest of this guide.

### Linux — libvirt / virt-install

```bash
virt-install \
  --connect qemu:///system \
  --import \
  --name tank-os \
  --memory 4096 \
  --disk /path/to/disk.qcow2 \
  --os-variant fedora-unknown \
  --cloud-init user-data=examples/cloud-init/openclaw-user-data.yaml,meta-data=examples/cloud-init/meta-data
```

### EC2 / Cloud

Use `examples/cloud-init/openclaw-user-data.yaml` as EC2 user data, replacing
the SSH key placeholder before launch. Connect with:

```bash
ssh openclaw@<ec2-public-ip>
```

## Step 6 — SSH Into The VM

```bash
ssh -i ~/.ssh/id_ed25519 -p 2222 openclaw@localhost
```

Adjust `-p 2222` to your actual forwarded port if different. For EC2 or a VM
with a direct IP, omit `-p` and use the host address.

Verify the service is healthy:

```bash
sudo -n true
sudo bootc status
systemctl --user status openclaw.service
podman ps
```

If `openclaw.service` fails with a permission error on `~/.openclaw`, fix
ownership and restart:

```bash
sudo chown -R openclaw:openclaw ~/.openclaw
systemctl --user restart openclaw.service
```

If the service times out pulling the container image, pull it manually first:

```bash
podman pull ghcr.io/openclaw/openclaw:latest
systemctl --user restart openclaw.service
```

## Step 7 — Configure Model Provider Keys

Create Podman secrets as the `openclaw` user, then sync and restart the service:

```bash
sudo -iu openclaw
printf '%s' "$ANTHROPIC_API_KEY"  | podman secret create anthropic_api_key -
printf '%s' "$OPENAI_API_KEY"     | podman secret create openai_api_key -
printf '%s' "$GEMINI_API_KEY"     | podman secret create gemini_api_key -
printf '%s' "$OPENROUTER_API_KEY" | podman secret create openrouter_api_key -
tank-openclaw-secrets
systemctl --user restart openclaw.service
```

Only create secrets for the providers you actually use. See
[model-providers.md](model-providers.md) for the full list and custom provider
setup.

## Step 8 — Configure service-gator (Optional)

service-gator gives OpenClaw agents scoped access to GitHub, GitLab, and other
services without exposing raw tokens.

```bash
sudo -iu openclaw
printf '%s' "$GH_TOKEN" | podman secret create gh_token -
$EDITOR ~/.config/service-gator/scopes.json
tank-openclaw-secrets
systemctl --user restart service-gator.service
```

See [service-gator.md](service-gator.md) for the scope file format.

## Step 9 — Access The OpenClaw Dashboard

Open an SSH tunnel from your local machine:

```bash
ssh -N \
  -i ~/.ssh/id_ed25519 \
  -p 2222 \
  -L 18789:127.0.0.1:18789 \
  -L 18790:127.0.0.1:18790 \
  openclaw@localhost
```

Then open `http://127.0.0.1:18789` in your browser.

To print the dashboard URL from the VM:

```bash
openclaw dashboard --no-open
```

## Post-Installation Reference

| Task | Command |
| --- | --- |
| Check gateway | `openclaw gateway status --deep` |
| Run diagnostics | `openclaw doctor` |
| View container logs | `podman logs -f openclaw` |
| Open container shell | `podman exec -it openclaw sh` |
| Edit OpenClaw config | `$EDITOR ~/.openclaw/openclaw.json` then `systemctl --user restart openclaw.service` |
| Upgrade the OS image | `sudo bootc upgrade --apply` |

## Troubleshooting

**Disk full on first boot** — Resize the QCOW2 before booting (Step 4).

**Service fails to start** — Check logs with `podman logs openclaw` and ensure
the OpenClaw container image pulled successfully.

**SSH key rejected** — Confirm the public key was injected at build time (Step 3
config) or via cloud-init, and that you are using the matching private key with
`-i`.

**Cannot find the forwarded SSH port (macOS)** — Run `ps aux | grep gvproxy` and
look for a `-ssh-port` argument, or use the Podman Desktop VM terminal directly.

For more detail, see the linked docs:

- [docs/build.md](build.md) — building the bootc container image and disk image
- [docs/provisioning.md](provisioning.md) — cloud-init, EC2, local VM access
- [docs/cli.md](cli.md) — the `openclaw` host CLI wrapper
- [docs/model-providers.md](model-providers.md) — configuring model provider keys
- [docs/service-gator.md](service-gator.md) — configuring service-gator
