# Verifying the reachability rules in an unfamiliar sandbox

The matrix in `SKILL.md` was measured on Docker Desktop with a macOS host. The
*rules* (§3 publishing semantics, §4 publishing rule) are Docker behaviour and
should hold anywhere; the *addresses* vary. When something contradicts the matrix,
re-measure instead of guessing — it takes about a minute.

## Re-measure the matrix

```sh
docker rm -f probe-a >/dev/null 2>&1
docker run -d --name probe-a -p 18080:80 nginx:alpine >/dev/null
sleep 2
IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' probe-a)
GW=$(docker network inspect bridge -f '{{(index .IPAM.Config 0).Gateway}}')
for t in "localhost:18080" "$IP:80" "$IP:18080" "$GW:18080" "host.docker.internal:18080"; do
  printf '%-38s -> %s\n' "$t" "$(timeout 4 curl -s -o /dev/null -w '%{http_code}' "http://$t" 2>/dev/null || echo FAIL)"
done
docker rm -f probe-a >/dev/null
```

Expected: `localhost` fails, `$IP:80` works, `$IP:18080` fails, gateway and
`host.docker.internal` work. Any deviation means this environment differs from the
documented one — trust the measurement and note it.

## Cross-network isolation (the subtle one)

Prove that *publishing*, not the network, decides cross-network reachability:

```sh
docker network create probe-net >/dev/null
docker run -d --name probe-unpub --network probe-net nginx:alpine >/dev/null
docker run -d --name probe-pub   --network probe-net -p 18082:80 nginx:alpine >/dev/null
sleep 2
for c in probe-unpub probe-pub; do
  ip=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $c)
  printf '%-12s %-14s -> %s\n' "$c" "$ip:80" \
    "$(timeout 4 curl -s -o /dev/null -w '%{http_code}' "http://$ip:80" 2>/dev/null || echo FAIL)"
done
docker rm -f probe-unpub probe-pub >/dev/null; docker network rm probe-net >/dev/null
```

Expected from a sandbox **not** attached to `probe-net`: `probe-pub` answers,
`probe-unpub` fails. Then `docker network connect probe-net "$(hostname)"` makes
both answer, by IP and by container name.

## Reading the host firewall (last resort)

Ground truth for a drop you cannot explain. Read-only, but needs a privileged
host-netns container — use sparingly and clean up:

```sh
docker run --rm --net=host --privileged --entrypoint sh alpine:latest -c \
  'apk add -q --no-cache nftables; nft list ruleset' | grep -E 'iifname|daddr'
```

What to look for, per target bridge `br-<id>` (get it from the network's ID —
`docker network inspect <net> -f '{{.Id}}'`, first 12 chars):

```
iifname != "br-X" oifname "br-X" ip daddr <container-ip> tcp dport <port> accept   # a published port
iifname != "br-X" oifname "br-X" drop                                              # everything else
iifname != "lo" ip daddr 127.0.0.1 tcp dport <published> drop                      # loopback-only publish
```

The presence or absence of that `accept` line is the whole explanation for
cross-network results. The `lo` rule is why a `-p 127.0.0.1:…` port fails from the
bridge gateway yet still answers on `host.docker.internal` under Docker Desktop:
the host-side forwarder delivers it via the VM's own loopback.

Older daemons express the same logic as iptables chains
(`iptables -S DOCKER-ISOLATION-STAGE-1/2`); newer ones use nftables and those
chains are empty or absent. Check both before concluding anything.

## Probing services you do not own

Read-only `curl` against another session's container is harmless and often the
fastest way to confirm the rules on live services. Never start, stop, restart, or
reconfigure a container you did not create, and never prune.
