# service-gator

`service-gator` is an MCP server that gives sandboxed agents scoped access to external services such as GitHub, GitLab, Forgejo/Gitea, and JIRA. It should run outside the OpenClaw gateway container so OpenClaw agents can use it without receiving raw PATs.

tank-os runs it as a second rootless Podman Quadlet owned by the `openclaw` user:

```text
/etc/containers/systemd/users/1000/service-gator.container
```

The service listens on loopback only:

```text
http://127.0.0.1:8080/mcp
```

## Configure Scopes

The first boot helper creates an empty scope file:

```text
~/.config/service-gator/scopes.json
```

Edit it as the `openclaw` user:

```bash
sudo -iu openclaw
$EDITOR ~/.config/service-gator/scopes.json
```

Example:

```json
{
  "scopes": {
    "gh": {
      "repos": {
        "myorg/myrepo": {
          "read": true,
          "push-new-branch": true,
          "create-draft": true
        }
      }
    }
  }
}
```

service-gator watches the scope file and can reload it without restarting.

## Create Secrets

Create secrets in the `openclaw` user's rootless Podman store before starting or restarting `service-gator.service`:

```bash
sudo -iu openclaw
printf '%s' "$GH_TOKEN" | podman secret create gh_token -
printf '%s' "$GITLAB_TOKEN" | podman secret create gitlab_token -
printf '%s' "$FORGEJO_TOKEN" | podman secret create forgejo_token -
printf '%s' "$JIRA_API_TOKEN" | podman secret create jira_api_token -
```

Then sync drop-ins and restart:

```bash
tank-openclaw-secrets
systemctl --user restart service-gator.service
```

service-gator receives secrets as files through Podman secrets and reads them via `*_FILE` variables such as `GH_TOKEN_FILE=/run/secrets/gh_token`.
The base Quadlet does not reference any secrets directly, so the service can start before credentials exist. `tank-openclaw-secrets` adds a user drop-in containing only the Podman secrets that exist in the `openclaw` user's rootless secret store.

## Connect OpenClaw

The exact OpenClaw MCP config shape should be finalized against the OpenClaw MCP config docs before baking a default into the image. The intended endpoint is:

```text
http://127.0.0.1:8080/mcp
```

Keep service-gator on loopback unless you intentionally put it behind gateway auth or another trusted edge.

For tools such as `git_push_local`, service-gator sees OpenClaw state mounted at `/workspaces`. Agent workspaces under `~/.openclaw/workspace-*` appear as `/workspaces/workspace-*` in the service-gator container.
