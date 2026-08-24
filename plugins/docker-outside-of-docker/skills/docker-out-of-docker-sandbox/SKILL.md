---
name: docker-out-of-docker-sandbox
description: Reach, start, and debug Docker containers and their network services from inside a sandboxed agent environment whose `docker` talks to the HOST daemon through a mounted socket (docker-outside-of-docker) — so containers you create are siblings that never share your localhost. Use ONLY in that setup — you are running inside a container (`/.dockerenv` exists), `docker info` succeeds, and you need to know whether a service is reachable, which address and port to use (container IP vs host gateway vs published port), how to publish or bind so something IS reachable, why `curl localhost` refuses even though the service is up, how `-v` bind-mount paths resolve, how to expose your own process to a container, or how to tell a macOS/Windows Docker Desktop host from a native Linux one. Triggers include a "connection refused" on localhost, "is the service up", "curl the app or API", "start a container", "run the compose stack", "the port isn't working", browser/E2E containers, and any port, host, or IP question inside such a sandbox. NOT for plain Docker on your own machine, and NOT for nested docker-in-docker (privileged dind).
---

# Docker from inside an agent sandbox (docker-outside-of-docker)

**Assume you are running inside a container, not on the developer's machine.** If a
Docker socket is present, the daemon is the **host's**: containers you create are
your **siblings**, not your children. Every surprise below follows from that.

Never answer a reachability question from memory. `localhost` is the single most
common wrong answer, and a refusal there is **no evidence the service is down**.

## 1. Confirm the setup (30 seconds, do it first)

```sh
ls /.dockerenv                       # exists -> you are in a container
docker info --format '{{.OperatingSystem}} / {{.ServerVersion}}'   # fails -> no Docker here; say so
docker inspect -f '{{.Name}} {{range $k,$v := .NetworkSettings.Networks}}{{$k}}={{$v.IPAddress}} {{end}}' "$(hostname)"
```

That last command printing your own container means you have socket access **and**
your own identity + IP — both needed below. If it fails, you cannot join networks
or learn your own address; fall back to host-gateway access only.

## 2. Identify the *host* OS — never from `uname`

`uname` describes your sandbox (always Linux), not the host.

| Signal | Meaning |
|---|---|
| `docker info` OperatingSystem = `Docker Desktop`, kernel `linuxkit` | VM-backed daemon (macOS/Windows host, or Docker Desktop on Linux) |
| OperatingSystem = a distro (`Ubuntu`, `Amazon Linux`, …) | native Linux daemon |
| `host.docker.internal` resolves (often `192.168.65.254`) | a host gateway name exists |
| your own bind-mount sources start with `/Users/…` | **macOS** host |
| … `/host_mnt/c/…` or `C:\` | **Windows** host |
| … `/home/…`, `/srv/…`, `/opt/…` | **Linux** host |

```sh
docker inspect -f '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}' "$(hostname)"
```

The mount table is the most reliable host-OS tell — and you need it for §5 anyway.

## 3. The core rule

**Port publishing (`-p`) happens in the *daemon host's* network namespace, not
yours.** A published port is therefore not on your `localhost`, and not on the
target container's own address either. Two ways in:

1. **Container IP + the port the service actually listens on** (the *container*
   port — publishing is irrelevant here).
2. **Host gateway + the *published* port** — `host.docker.internal`, else the
   bridge gateway (`docker network inspect bridge -f '{{(index .IPAM.Config 0).Gateway}}'`,
   typically `172.17.0.1`).

## 4. Reachability matrix

Measured from a sandbox on the default `bridge`; target published `-p 18080:80`:

| From the sandbox, to… | Result |
|---|---|
| `localhost:18080` / `127.0.0.1:18080` | **fail** — your own namespace |
| `<container-ip>:80` (container port), same network | **works** |
| `<container-ip>:18080` (published port) | **fail** — that port doesn't exist inside |
| `172.17.0.1:18080` (bridge gateway) | **works** |
| `host.docker.internal:18080` | **works** |
| gateway, when published `-p 127.0.0.1:18081:80` | **fail** — loopback-bound |
| `host.docker.internal:18081` (same loopback publish) | **works, Docker Desktop only** |
| `<container-ip>:80`, **another** network, port **published** | **works** |
| `<container-ip>:80`, another network, **not** published | **fail** — firewall drop |
| `<name>:80` before joining that network | **fail** — DNS is per-network |
| `<name>:80` after `docker network connect` | **works** |

**The publishing rule (non-obvious).** Docker writes a firewall accept rule per
*published* port keyed on **container IP + container port**, accepting traffic
from any other bridge; everything else to that bridge is dropped. So a container
on a network you have **not** joined is reachable at its container IP **only for
ports it published** — and the publish *bind address is irrelevant* to this path
(`-p 127.0.0.1:8111:8000` still accepts `<container-ip>:8000` cross-network).
Inside your own network, ICC allows everything and publishing does not matter.

## 5. Deciding how to reach a service

```
Is the service in a sibling container?
├─ yes ─ Do we share a network?  (compare .NetworkSettings.Networks on both)
│        ├─ yes → http://<container-ip>:<CONTAINER port>        ← most reliable
│        │        or http://<container-name>:<port> on a user-defined network
│        └─ no  → does it publish that port?  (docker port <c>)
│                 ├─ yes → http://<container-ip>:<CONTAINER port>, or
│                 │        http://host.docker.internal:<PUBLISHED port>
│                 └─ no  → join its network (§6). Nothing else reaches it.
└─ no ── A process on the host itself → http://host.docker.internal:<port>
         (native Linux without that name: the bridge gateway)
```

Prefer **container IP + container port**: immune to how ports were published,
identical on macOS, Windows, and Linux hosts. Read bindings with
`docker port <c>` or `docker ps --format '{{.Names}}\t{{.Ports}}'` —
`0.0.0.0:`/`[::]:` means the gateway works, `127.0.0.1:` means it does not.

## 6. Joining a network (the reliable fix)

```sh
docker network ls
docker network connect <network> "$(hostname)"     # DNS-by-container-name now works
# ... http://<service-name>:<port> ...
docker network disconnect <network> "$(hostname)"  # clean up when done
```

Best way to talk to a Compose stack: join the project network, address services
by **service name** and **internal** port.

## 7. Bind mounts resolve on the *host* filesystem

`-v /path:/dest` is interpreted by the daemon, so `/path` must exist **on the
host**. A path that exists only inside your sandbox (a scratch/temp dir) mounts as
an **empty directory** — silently, no error. Get real host paths from your own
mount table (§2). Many sandboxes mount the project at an identical path on both
sides, which is why relative paths appear to work; never assume it.

## 8. Listener pitfalls

- A service bound to `127.0.0.1` **inside** its container is unreachable from
  anywhere, published or not:
  `docker exec <c> sh -c 'netstat -ltn 2>/dev/null || ss -ltn'` — you want
  `0.0.0.0:<port>`. Same rule for anything you start: bind `0.0.0.0`.
- Publish on all interfaces (`-p 8080:80`) when you want gateway access;
  `-p 127.0.0.1:8080:80` defeats it.

## 9. Exposing *your own* process to containers

You were not started with `-p`, so nothing you run is published and
`host.docker.internal:<your-port>` will fail. Siblings on your network reach you
at **your** container IP:

```sh
MYIP=$(docker inspect -f '{{.NetworkSettings.Networks.bridge.IPAddress}}' "$(hostname)")
# from a sibling: curl http://$MYIP:<port>
```

`--network host` on a container you start means the **daemon host's** namespace,
not yours — it does not expose your ports.

## 10. Compose

```sh
docker compose ps                          # empty = nothing running
docker compose port <service> <port>       # the actual published binding
docker network ls | grep <project>         # then §6 to join it
docker compose run --rm <service> <cmd>    # one-off, no networking tricks needed
```

Run these **from the directory holding the compose file**; otherwise you get
`no configuration file provided: not found`, which means wrong directory, not
"nothing running".

## 11. Native Linux host: what changes

- `host.docker.internal` may not resolve → use the bridge gateway, or pass
  `--add-host=host.docker.internal:host-gateway` to containers you create.
- A `127.0.0.1`-published port is genuinely unreachable from containers (no
  Docker Desktop forwarder) → use the container IP.
- Bind-mount paths are literal; no `/Users` or `/host_mnt` translation.
- Container IPs are reachable from the host too (not true on Docker Desktop).

## 12. Hygiene — siblings outlive you

They belong to the host daemon, not your session, and you are one of several
tenants on it.

- Name what you create (`--name <prefix>-<purpose>`) so you can find it again.
- Remove it when done: `docker rm -f`, `docker network rm`, and disconnect any
  network you joined. Leaks hold ports and confuse the next session.
- Never prune globally, never kill containers you did not start, never remap a
  shared project's ports. A taken port usually means a stack is already up — use
  it.

## 13. Diagnostics

```sh
docker inspect -f '{{.Name}}{{range $k,$v := .NetworkSettings.Networks}} {{$k}}={{$v.IPAddress}}{{end}}' \
  $(docker ps -q) 2>/dev/null     # who is on which network at which IP
docker ps --format '{{.Names}}\t{{.Ports}}'   # what is published
```

Between those two you can answer almost any reachability question here. If
something still behaves inexplicably, see `references/verification.md` for
reading the host firewall and re-measuring the matrix in an unfamiliar sandbox.
