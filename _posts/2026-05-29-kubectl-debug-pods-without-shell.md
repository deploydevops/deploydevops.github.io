---
title: "kubectl debug: how to debug pods that have no shell"
date: 2026-05-29
tags: [kubernetes, debugging, kubectl]
read_time: 4
description: "Your pod runs a distroless image with no /bin/sh. kubectl exec fails. Here's how to attach a debug container and get full access."
---

You're on call. Something is wrong with a pod. You try the obvious thing:

```bash
kubectl exec -it api-server-7d4f8b6c9-k2xnp -- sh
```

And you get this:

```
error: failed to exec in container: failed to start exec:
OCI runtime exec failed: exec: "/bin/sh":
stat /bin/sh: no such file or directory
```

The image is distroless. There is no shell. No bash, no sh, no busybox. This is how production images should be built, and it's great until you need to debug one.

## The fix: kubectl debug

`kubectl debug` attaches an ephemeral container to your running pod. The debug container shares the same namespaces (network, PID, IPC) as the target container, so you can inspect everything without modifying the original image.

```bash
kubectl debug -it api-server-7d4f8b6c9-k2xnp \
  --image=busybox \
  --target=api-server
```

That's it. You get a shell inside a busybox container that can see the same network interfaces and processes as your app.

## What you can do once you're in

**See processes:**

```bash
ps aux
# PID   USER     COMMAND
#     1 nonroot  /app/api-server --port=8080
#    47 root     /bin/sh
```

**Check network:**

```bash
netstat -tlnp
# tcp  0.0.0.0:8080  LISTEN  1/api-server
# tcp  127.0.0.1:6379  LISTEN  -
```

**Test endpoints from inside the pod:**

```bash
wget -qO- localhost:8080/health
# {"status":"healthy","uptime":"3d2h","connections":142}
```

**Read environment variables (even if the container doesn't have `env`):**

```bash
cat /proc/1/environ | tr '\0' '\n' | grep DB
# DB_HOST=redis-cache-0.redis.default.svc.cluster.local
# DB_PORT=6379
```

`/proc/1/environ` gives you the environment of PID 1 (the main process) regardless of what tools are installed in the target container. I use this one constantly.

## Choosing the right debug image

Busybox works for most cases, but if you need more tools:

- `busybox` — basic unix tools, wget, ps, netstat. 1.5MB.
- `alpine` — busybox + apk package manager. 7MB. Good when you need to install something on the fly.
- `nicolaka/netshoot` — curl, tcpdump, nmap, iperf, dig, nslookup. 300MB. Overkill for most situations but exactly what you want when you're chasing a network issue.

```bash
# For network issues specifically:
kubectl debug -it my-pod --image=nicolaka/netshoot --target=app

# Then inside:
tcpdump -i eth0 port 8080
curl -v http://other-service:3000/health
dig redis.default.svc.cluster.local
```

## What happens when you exit

The ephemeral container is removed automatically. You don't need to clean anything up.

If you need to check what debug containers are attached to a pod:

```bash
kubectl describe pod api-server-7d4f8b6c9-k2xnp | grep -A5 "Ephemeral"
```

## Requirements

Ephemeral containers have been GA since Kubernetes 1.25 (August 2022). The `EphemeralContainers` feature gate is on by default since then, so unless someone explicitly turned it off, you're good.

## The command to bookmark

```bash
kubectl debug -it <pod-name> --image=busybox --target=<container-name>
```

I had to google this syntax the last two times I needed it at 3am, so now it lives here.
