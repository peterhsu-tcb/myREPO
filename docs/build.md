# Build

## Build The Bootc Container Image

Skip this section if you want to build a disk image directly from the published
image:

```text
quay.io/sallyom/tank-os:latest
```

That image is published for both `arm64` and `amd64`, so Podman Desktop or
bootc-image-builder can select the right architecture for your target.

Build the bootc image from the repo root. In these commands, the final `bootc`
argument is the build context directory in this repo:

```text
tank-os/
├── bootc/
│   ├── Containerfile
│   └── rootfs/
└── docs/
```

For Apple Silicon:

```bash
podman build \
  --platform linux/arm64 \
  -t localhost/tank-os:latest \
  -f bootc/Containerfile \
  bootc
```

For x86_64:

```bash
podman build \
  --platform linux/amd64 \
  -t localhost/tank-os:latest \
  -f bootc/Containerfile \
  bootc
```

The default base is `quay.io/fedora/fedora-bootc:latest`. For a pinned build:

```bash
podman build \
  --build-arg FEDORA_BOOTC_BASE=quay.io/fedora/fedora-bootc:<tag> \
  -t localhost/tank-os:<tag> \
  -f bootc/Containerfile bootc
```

## Build A Disk Image With Podman Desktop

The Podman Desktop BootC extension can build a VM disk image from
`localhost/tank-os:latest` or the published `quay.io/sallyom/tank-os:latest`.

Recommended local test settings on Apple Silicon:

- Bootc image: `localhost/tank-os:latest`
- Or published image: `quay.io/sallyom/tank-os:latest`
- Disk image type: `qcow2`
- Target architecture: `arm64` or `aarch64`
- Root filesystem: `xfs`
- Output folder: a dedicated writable directory such as `/Users/<you>/git/out-tank-os`
- User: `openclaw`
- SSH public key: your Mac SSH public key
- Groups: `wheel`
- Password: leave empty

The output should be:

```text
<output-folder>/qcow2/disk.qcow2
```

See:

- Podman Desktop BootC extension: https://github.com/podman-desktop/extension-bootc
- bootc-image-builder docs: https://osbuild.org/docs/bootc/
- bootc docs: https://bootc-dev.github.io/bootc/

## Build A Disk Image Manually

Create an output directory:

```bash
mkdir -p out-tank-os
```

Optionally create a bootc-image-builder config to inject a local SSH key. This
is convenient for local VM tests. Do not put private keys or long-lived secrets
here.

```bash
cat > out-tank-os/config.json <<'EOF'
{
  "customizations": {
    "user": [
      {
        "name": "openclaw",
        "key": "ssh-ed25519 REPLACE_WITH_YOUR_PUBLIC_KEY tank-os",
        "groups": ["wheel"]
      }
    ]
  }
}
EOF
```

Build the QCOW2 with bootc-image-builder. On macOS with Podman Desktop, use the
rootful Podman machine connection because bootc-image-builder needs privileged
access to the container storage.

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

For x86_64 output, use:

```bash
--target-arch amd64
```

The resulting disk image is:

```text
out-tank-os/qcow2/disk.qcow2
```

## What The Image Installs

The image creates an `openclaw` login user with UID/GID 1000, enables linger for that user, and installs a rootless Quadlet at:

```text
/etc/containers/systemd/users/1000/openclaw.container
```

On boot, OpenClaw state lives at:

```text
/var/home/openclaw/.openclaw
```

When logged in as `openclaw`, that is `~/.openclaw`.

## Upgrade A Running VM

After pushing a new bootc image, switch the VM to the registry ref:

```bash
sudo bootc status
sudo bootc switch --apply quay.io/sallyom/tank-os:latest
```

After the reboot, future updates against the same tracked tag can use:

```bash
sudo bootc upgrade --apply
```
