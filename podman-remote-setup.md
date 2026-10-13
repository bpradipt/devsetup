# Rootless Podman remote for local agents

This guide configures a local podman API for an agent restricted by Landlock, such as an agent under nono.

It contains the required commands and configuration.

The agent uses a Unix socket. A separate, unprivileged account runs podman outside the agent sandbox.
A systemd mount namespace limits the host directories that Podman can use as bind-mount sources.

```text
agent and remote client under Landlock
  -> /run/podman-dev/podman.sock
  -> rootless Podman as podman-dev, inside a restricted mount namespace
  -> containers with access to /srv/work
```

## Security boundary

The API permits arbitrary code execution as its service account. Socket access therefore grants control of that account's containers and storage.
Podman provides no per-request authorization for this API. See the [Podman service security documentation](https://docs.podman.io/en/latest/markdown/podman-system-service.1.html#security).

The agent's Landlock restrictions do not transfer to the service. The service resolves bind-mount paths using its own permissions and namespace.
A service under the developer's account can expose files that Landlock denies to the agent.

This setup prevents the agent from reading host files that are (a) outside /srv/work, (b) not world-readable,
(c) not under the bound system directories (/etc, /usr, /bin, /sbin, /lib, /lib64), and 
(d) not served by any local socket/API the agent can still reach. 

It accepts host-network, host-PID/IPC, privileged-container, and client-seccomp risk.


This configuration uses four layers:

| Layer | Restriction |
|---|---|
| Service account | Separate UID, no login shell, no supplementary groups |
| Host file access | Workspace ACLs, no service grant on the developer home |
| Service mount namespace | Empty root, explicit system and workspace binds, private temporary directories |
| Agent socket access | Exact socket grant and pathname Unix socket mediation |

The intended boundary covers private host files outside the approved workspace.
The service also needs its own storage, runtime directories, and readable system files.
This is not a guarantee that every service-readable byte is directly readable by the agent.

## Requirements and example paths

Use a Linux host with systemd, enabled unprivileged user namespaces, and Landlock.
Use a kernel with Landlock ABI v6 (Linux 6.12 or newer) for abstract Unix socket scoping.
Require full enforcement of the requested sandbox restrictions at agent startup.

Install these host packages through your distribution:

- Podman server at `/usr/bin/podman`, an OCI runtime, and the distribution's container configuration and seccomp profile.
- `newuidmap` and `newgidmap`, usually from `uidmap` or `shadow-utils`.
- `fuse-overlayfs` and a supported rootless network helper, such as `pasta` or `slirp4netns`.
- ACL tools (`setfacl` and `getfacl`), Python 3, and systemd tools.
- A Podman remote client and nono with `linux.af_unix_mediation` support.

Use a 6.x or newer client and verify your client/server pair.
Install a trusted release through your distribution or the [official Podman project](https://github.com/containers/podman/releases).

| Name | Example |
|---|---|
| Developer account | `devuser` — replace this with the account that runs the agent |
| Service account | `podman-dev` |
| Dedicated client group | `podman-agent` |
| Shared workspace | `/srv/work`, outside the developer home |
| API socket | `/run/podman-dev/podman.sock` |

Run host administration outside the agent sandbox. The commands configure the host; they do not repair sandbox denials.
Use a fresh service account and a workspace reserved for this purpose.
For an existing deployment, review its units, drop-ins, account groups, storage, and workloads before replacing configuration.

## 1. Create the service account and workspace

Run this block in Bash outside the sandbox. Replace `devuser` first.

```bash
set -euo pipefail
CLIENT=devuser
SVC=podman-dev
SOCKET_GROUP=podman-agent
WORKSPACE=/srv/work

CLIENT_UID=$(id -u "$CLIENT")
CLIENT_GROUP=$(id -gn "$CLIENT")
CLIENT_HOME=$(getent passwd "$CLIENT" | cut -d: -f6)
test "$CLIENT_UID" -ne 0
case "$WORKSPACE" in
  "$CLIENT_HOME"|"$CLIENT_HOME"/*)
    echo 'Choose a workspace outside the developer home.' >&2
    exit 1
    ;;
esac

if ! id "$SVC" >/dev/null 2>&1; then
  sudo useradd --create-home --shell /usr/sbin/nologin "$SVC"
fi
SVC_UID=$(id -u "$SVC")
SVC_HOME=$(getent passwd "$SVC" | cut -d: -f6)
test "$SVC_UID" -ne 0
test "$SVC_UID" -ne "$CLIENT_UID"
id "$SVC"

# Require subordinate IDs allocated by the host administrator or useradd.
awk -F: -v user="$SVC" '$1 == user && $3 >= 65536 {ok=1} END {exit !ok}' /etc/subuid
awk -F: -v user="$SVC" '$1 == user && $3 >= 65536 {ok=1} END {exit !ok}' /etc/subgid

getent group "$SOCKET_GROUP" >/dev/null || sudo groupadd "$SOCKET_GROUP"
sudo usermod -aG "$SOCKET_GROUP" "$CLIENT"
sudo chmod 0700 "$SVC_HOME"
sudo loginctl enable-linger "$SVC"
sudo systemctl start "user@$SVC_UID.service"
test -d "/run/user/$SVC_UID"

sudo mkdir -p "$WORKSPACE"
sudo chown "$CLIENT:$CLIENT_GROUP" "$WORKSPACE"
sudo chmod 0770 "$WORKSPACE"
sudo setfacl -R -m "u:$SVC:rwX,o::---" "$WORKSPACE"
sudo setfacl -R -d -m "u:$SVC:rwX,o::---" "$WORKSPACE"
getfacl "$WORKSPACE"
```

If either subordinate-ID check fails, allocate unused ranges through the host's account-management tools before you continue.
Allocate at least 65,536 subordinate UIDs and GIDs. Check all existing allocations for overlap.
Do not copy a fixed range from another host. See the [rootless Podman prerequisites](https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md).

Confirm that `podman-dev` has only its primary group. Keep it out of administrative, device-access, and container-runtime groups.
Confirm that `podman-agent` contains only the intended developer account.
Log out and back in after the group change. A new terminal alone does not update existing login credentials.

Keep the workspace outside the developer home. Do not grant the service an ACL on that home, including traverse-only access.
Default ACLs apply to new files. A tool that creates mode `0700` directories or `0600` files can remove effective service access.
Inspect the ACL mask when a workspace mount fails. Restrict any ACL correction to the shared workspace.

## 2. Configure rootless Podman storage and cgroups

Continue in the same host Bash shell.

```bash
sudo -u "$SVC" mkdir -p "$SVC_HOME/.config/containers/containers.conf.d"
sudo -u "$SVC" tee "$SVC_HOME/.config/containers/containers.conf.d/90-agent-cgroupfs.conf" >/dev/null <<'EOF'
[engine]
cgroup_manager = "cgroupfs"
EOF
```

`MountAPIVFS=yes` hides the host systemd socket from the service. This setup therefore uses `cgroupfs`.
Rootless cgroupfs can have limited resource-control support. Verify resource limits before relying on them.
Keep storage in the service home. Remove custom storage paths outside that home from an existing service configuration.

## 3. Install the restricted socket and service

The dedicated group controls client access. The socket has mode `0660`, and its parent has mode `0750`.
The root-owned parent prevents clients from replacing the socket path.

Run these commands in the same host Bash shell:

```bash
sudo tee /etc/tmpfiles.d/podman-dev.conf >/dev/null <<EOF
d /run/podman-dev 0750 root $SOCKET_GROUP -
EOF
sudo systemd-tmpfiles --create /etc/tmpfiles.d/podman-dev.conf

sudo tee /etc/systemd/system/podman-dev.socket >/dev/null <<EOF
[Unit]
Description=Restricted local rootless Podman API socket
After=systemd-tmpfiles-setup.service

[Socket]
ListenStream=/run/podman-dev/podman.sock
SocketUser=$SVC
SocketGroup=$SOCKET_GROUP
SocketMode=0660
DirectoryMode=0750
RemoveOnStop=yes

[Install]
WantedBy=sockets.target
EOF

sudo tee /etc/systemd/system/podman-dev.service >/dev/null <<EOF
[Unit]
Description=Rootless Podman API in a restricted mount namespace
Requires=podman-dev.socket user@$SVC_UID.service
After=podman-dev.socket user@$SVC_UID.service

[Service]
User=$SVC
Environment=HOME=$SVC_HOME
Environment=XDG_RUNTIME_DIR=/run/user/$SVC_UID
ExecStart=/usr/bin/podman system service --time=0
TemporaryFileSystem=/:ro
MountAPIVFS=yes
BindReadOnlyPaths=/usr /etc /bin /sbin /lib /lib64
BindPaths=$SVC_HOME $WORKSPACE /run/podman-dev /run/user/$SVC_UID
PrivateTmp=yes
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now podman-dev.socket
```

The system manager creates the namespace, then runs Podman as the nonzero service UID.
This is rootless Podman even though root installs and manages the system units.
The inherited socket provides the endpoint. Do not add a TCP listener.

The namespace hides other host homes, `/opt`, `/boot`, `/var/log`, and other unlisted data directories.
System directories remain readable, subject to service-account permissions. `/etc` must not contain secrets readable by that account.
Private `/tmp` and `/var/tmp` prevent ordinary access to host temporary files.
Review any additional bind path as an expansion of the service's access.
See the [systemd namespace directives](https://github.com/systemd/systemd/blob/main/man/systemd.exec.xml).

On distributions without `/lib64`, remove that path from `BindReadOnlyPaths` after checking the host layout.
Keep setuid `newuidmap` and `newgidmap` usable.
Do not add `NoNewPrivileges=yes` or hardening that implies it, such as the existing setup's `SystemCallFilter` or `RestrictNamespaces` options.
Such changes can prevent rootless Podman startup.

## 4. Disable the developer's alternate Podman service

Run this command as the developer, outside the sandbox:

```bash
systemctl --user mask --now podman.socket podman.service
```

Confirm that no developer-owned Podman API process still listens on an old socket.
Masking units does not stop a separately started `podman system service` process.
Do not enable a developer-owned or rootful API for the agent.
The sandbox must deny Docker sockets, other Podman sockets, session D-Bus, systemd control sockets, and credential-agent sockets.

## 5. Configure the agent sandbox

Landlock filesystem grants alone do not mediate pathname Unix socket connections.
Nono adds separate mediation through `linux.af_unix_mediation: "pathname"` and explicit `filesystem.unix_socket` grants.
Its mediation uses seccomp user notification. Abstract sockets need separate Landlock scoping.
See [nono's Landlock and Unix socket documentation](https://github.com/nolabs-ai/nono/blob/main/docs/cli/internals/landlock.mdx).

Use a nono version that supports these fields. The source setup used nono 0.79.
For another Landlock launcher, require equivalent pathname socket mediation and abstract socket restrictions.

Create a profile draft outside the sandbox with a text editor:

```bash
mkdir -p "$HOME/.config/nono/profile-drafts"
${EDITOR:-vi} "$HOME/.config/nono/profile-drafts/agent-podman.json"
```

Use this JSON. Replace `YOUR_EXISTING_AGENT_PROFILE` with the profile you already use for the agent.
Keep that profile's credential protections and required agent runtime grants.

```json
{
  "extends": "YOUR_EXISTING_AGENT_PROFILE",
  "meta": {
    "name": "agent-podman",
    "description": "Local rootless Podman through one restricted Unix socket"
  },
  "workdir": { "access": "readwrite" },
  "linux": { "af_unix_mediation": "pathname" },
  "security": { "ipc_mode": "shared_memory_only" },
  "filesystem": {
    "unix_socket": ["/run/podman-dev/podman.sock"],
    "allow": ["/srv/work", "/tmp/agent-podman-$UID"],
    "read": ["/etc/containers", "$HOME/.local/bin"]
  },
  "environment": {
    "deny_vars": [
      "SSH_AUTH_SOCK", "DBUS_SESSION_BUS_ADDRESS",
      "DISPLAY", "WAYLAND_DISPLAY", "XAUTHORITY"
    ],
    "set_vars": {
      "CONTAINER_HOST": "unix:///run/podman-dev/podman.sock",
      "TMPDIR": "/tmp/agent-podman-$UID",
      "XDG_RUNTIME_DIR": "/tmp/agent-podman-$UID"
    }
  }
}
```

Add a read grant for the remote-client installation directory if it differs from the example.
The `$UID` refers to the developer, not the service account.

Validate and promote the draft outside the sandbox:

```bash
nono profile validate --strict --draft agent-podman
nono profile promote agent-podman
nono profile show agent-podman
install -d -m 0700 "/tmp/agent-podman-$(id -u)"
cd /srv/work
nono run --dry-run -v --profile agent-podman -- YOUR_AGENT_COMMAND
```

Inspect the resolved profile and launch capabilities. Inheritance and launcher capabilities can add grants.
Require the exact Podman socket grant. Remove inherited grants for other service sockets and broad socket directories.
Do not assume that the child profile's socket array removes inherited grants.
Do not grant the service home or its runtime directory to the agent.
Stop if the launcher cannot enforce the requested socket or IPC restrictions.

After review, replace `YOUR_AGENT_COMMAND` with the agent executable and start it:

```bash
nono run -v --profile agent-podman -- YOUR_AGENT_COMMAND
```

## 6. Use the remote client

Inside the sandbox, use the remote option:

```bash
podman --remote --url unix:///run/podman-dev/podman.sock info
```

`CONTAINER_HOST` selects the same endpoint for clients that honor it. An environment variable is convenience, not an access restriction.
The socket permissions and sandbox policy enforce the endpoint restriction.

Bind-mount sources are paths on the service host. Put container inputs under `/srv/work`.
The service's private `/tmp` does not contain the agent's `$TMPDIR`.
Build contexts and image archives uploaded by the client must also be readable under the agent's Landlock grants.

## 7. Verify the boundary

Outside the sandbox, inspect the host configuration:

```bash
id podman-dev
getent group podman-agent
stat -c '%a %U:%G %n' /run/podman-dev /run/podman-dev/podman.sock
getfacl /srv/work
getfacl "$HOME"
sudo systemctl cat podman-dev.service podman-dev.socket
```

Require directory mode `750` with owner `root:podman-agent`, and socket mode `660` with owner `podman-dev:podman-agent`.
Require no service ACL on the developer home. Inspect effective unit drop-ins for extra binds or weakened restrictions.

Run the next block in Bash inside the sandbox, as the developer.
Use a trusted image. The example image pull requires service-side registry access.

```bash
set -euo pipefail
PODMAN_URL=unix:///run/podman-dev/podman.sock
IMAGE=docker.io/library/alpine:3.21
pr() { podman-remote --url "$PODMAN_URL" "$@"; }
# For a full Podman client instead:
# pr() { podman --remote --url "$PODMAN_URL" "$@"; }

pr info --format '{{.Host.Security.Rootless}}'
pr run --rm "$IMAGE" cat /proc/self/uid_map
pr run --rm "$IMAGE" true

PROBE=$(mktemp -d /srv/work/podman-probe.XXXXXX)
trap 'rm -rf "$PROBE"' EXIT
chmod 0770 "$PROBE"  # mktemp uses 0700, which masks the inherited service ACL.
printf 'workspace-ok\n' > "$PROBE/input"
pr run --rm --network=none -v "$PROBE:/work:rw" "$IMAGE" \
  sh -c 'cat /work/input; printf "container-ok\n" > /work/output'
test "$(cat "$PROBE/output")" = container-ok

# --mount does not create a missing bind source.
if pr run --rm --network=none --mount "type=bind,src=$HOME,dst=/probe,ro" "$IMAGE" true; then
  echo 'FAIL: developer home is reachable' >&2
  exit 1
fi
if pr run --rm --network=none --privileged --mount "type=bind,src=$HOME,dst=/probe,ro" "$IMAGE" true; then
  echo 'FAIL: privileged container reaches developer home' >&2
  exit 1
fi
ln -s "$HOME" "$PROBE/home-link"
if pr run --rm --network=none --mount "type=bind,src=$PROBE/home-link,dst=/probe,ro" "$IMAGE" true; then
  echo 'FAIL: workspace symlink reaches developer home' >&2
  exit 1
fi
```

Require `Rootless=true`. Require container UID zero to map to the host UID of `podman-dev`, distinct from the developer UID.
Require the ordinary container and workspace read/write probe to succeed.
For each denied mount, inspect the error. Require a missing or inaccessible source, rather than an image or container startup error.
A failed ordinary container is not evidence of isolation.

Also verify these properties before accepting the deployment:

- A known host file in an unbound data directory cannot be read through a container mount.
- A host temporary marker does not appear through the service's `/tmp` view.
- A harmless listening test socket outside the grant is denied inside nono but reachable outside it.
- The resolved policy grants no other container-runtime or privileged control socket.

Repeat the checks after changes to Podman, nono, the kernel, unit configuration, or sandbox grants.
These probes check the intended paths. They are not proof against every kernel or IPC attack.

## Limits and operating rules

Containers execute outside the agent's Landlock policy. Their network requests and image pulls also originate outside that policy.
The API allows host networking, host PID namespaces, privileged containers, and client-selected seccomp profiles.
Do not treat a client command convention or default seccomp profile as an enforced API policy.

Host networking can expose loopback services and the LAN. Host PID access can expose process arguments and additional `/proc` interfaces.
Nono documents a race limitation in pathname socket mediation for hostile multithreaded processes.
Use this setup only where those residual capabilities fit the threat model.
For stronger requirements, add independently enforced service-side controls or a stronger outer isolation boundary.

Keep the service UID dedicated to Podman. Do not run unrelated services or credential holders under that UID.
Treat the service home and all shared workspace content as controlled by API clients.
Do not execute service-owned files outside the restricted service namespace.
Review workspace scripts before you run them outside the agent sandbox.

One socket shares one container store across sessions. Containers can outlive the agent.
Remove only resources that belong to your session. Separate trust domains need separate service accounts, storage, workspaces, and sockets.

## Troubleshooting

For service startup errors, inspect logs outside the sandbox:

```bash
sudo systemctl status podman-dev.socket podman-dev.service
sudo journalctl -u podman-dev.service -n 100 --no-pager
```

Check subordinate IDs, the service runtime directory, `newuidmap`, and namespace bind paths.
A socket file alone does not prove that the service works.

For a client-side socket denial under nono, query the active policy:

```bash
nono why --self --path /run/podman-dev/podman.sock --op read
```

This command queries filesystem access. It does not independently test pathname socket mediation.
Inspect the explicit socket grants and the connection error as well.
Only a successful `DENIED` result, or an error explicitly naming nono as the denying sandbox, confirms a nono boundary.
Otherwise report: `Nono sandbox unconfirmed; this permission failure needs a non-sandbox diagnosis.`
Distinguish a client socket denial from a service-side bind-mount denial.
A denied developer-home mount is the intended boundary. Move approved inputs into the workspace.
Nono cannot add capabilities to an already restricted process. A corrected socket profile requires an agent restart.

## Stop the API

After reviewing active workloads, stop new API access outside the sandbox:

```bash
sudo systemctl disable --now podman-dev.socket
sudo systemctl stop podman-dev.service
```

These commands preserve the service account, container storage, and workspace.
They do not guarantee that existing containers stop. Review and stop your workloads separately before removing account or storage data.

## TODO: service-side API filtering for containment

Podman exposes no per-request authorisation, so any containment beyond the mount-namespace file boundary must come from an external mediator,
not from client command conventions. 
A future hardening step is a filtering proxy in front of podman.sock that parses each create/run payload and rejects disallowed options
before forwarding to the real socket. The agent connects to the proxy socket; the proxy holds the only grant on the podman socket.
