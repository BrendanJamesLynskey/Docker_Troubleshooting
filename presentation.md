# Docker Troubleshooting

**Diagnose, Debug, and Resolve Container Issues**

> A systematic approach to diagnosing and resolving the most common Docker issues -- from container crashes to networking failures.

`Observe` -> `Diagnose` -> `Isolate` -> `Fix` -> `Verify`

Debugging - Logs - Monitoring - DevOps

---

## Table of Contents

1. [Topics](#topics)
2. [Troubleshooting Methodology](#troubleshooting-methodology)
3. [Container Lifecycle & Exit Codes](#container-lifecycle--exit-codes)
4. [Docker Logs & Log Drivers](#docker-logs--log-drivers)
5. [docker exec for Live Debugging](#docker-exec-for-live-debugging)
6. [docker inspect Deep Dive](#docker-inspect-deep-dive)
7. [docker stats & Resource Monitoring](#docker-stats--resource-monitoring)
8. [Debugging Build Failures](#debugging-build-failures)
9. [Layer Caching Issues](#layer-caching-issues)
10. [Networking Problems](#networking-problems)
11. [Volume & Permission Issues](#volume--permission-issues)
12. [Container Keeps Restarting](#container-keeps-restarting)
13. [Image Pull Errors & Registry Issues](#image-pull-errors--registry-issues)
14. [Disk Space Problems](#disk-space-problems)
15. [Performance Debugging](#performance-debugging)
16. [Docker Daemon Logs](#docker-daemon-logs)
17. [Useful Debugging Tools](#useful-debugging-tools)
18. [Troubleshooting Cheat Sheet](#troubleshooting-cheat-sheet)
19. [Summary & Further Reading](#summary--further-reading)

---

## Topics

### Methodology & Fundamentals

- Troubleshooting methodology
- Container lifecycle & exit codes
- Docker logs & log drivers
- docker exec for live debugging

### Inspection & Monitoring

- docker inspect deep dive
- docker stats & resource monitoring
- Performance debugging (CPU, mem, I/O)
- Docker daemon logs

### Build & Image Issues

- Debugging build failures
- Layer caching issues
- Image pull errors & registry issues
- Disk space problems & pruning

### Runtime & Networking

- Networking problems (DNS, ports)
- Volume & permission issues
- Container restart loops
- Debugging tools & cheat sheet

---

## Troubleshooting Methodology

A systematic approach prevents random guessing and saves hours of debugging time.

`Reproduce` -> `Observe` -> `Hypothesize` -> `Test` -> `Fix`

### Step 1: Gather Information

- Check container status: `docker ps -a`
- Read the logs: `docker logs <container>`
- Inspect configuration: `docker inspect`
- Check resource usage: `docker stats`

### Step 2: Isolate the Problem

- Is it a build issue or runtime issue?
- Does it fail locally or only in CI/prod?
- Is it the app, the image, or the host?
- Did anything change recently?

### The Golden Rule

Always check `docker logs` first -- 80% of container issues are visible in the application logs before you need any other tool.

---

## Container Lifecycle & Exit Codes

`Created` -> `Running` -> `Paused` -> `Exited` -> `Dead`

| Exit Code | Meaning | Common Cause |
|-----------|---------|--------------|
| 0 | Normal exit | Process completed successfully |
| 1 | Application error | Unhandled exception, config error, missing file |
| 126 | Permission denied | Entrypoint/CMD not executable |
| 127 | Command not found | Binary missing from image, wrong PATH |
| 137 | SIGKILL (128+9) | OOM killed or `docker kill` |
| 139 | SIGSEGV (128+11) | Segmentation fault in application |
| 143 | SIGTERM (128+15) | Graceful stop via `docker stop` |

```bash
# Check exit code of a stopped container
docker inspect --format='{{.State.ExitCode}}' my-container

# See full state including OOMKilled flag
docker inspect --format='{{json .State}}' my-container | jq .
```

---

## Docker Logs & Log Drivers

Container logs capture everything written to **stdout** and **stderr** by the main process (PID 1).

```bash
# Basic log viewing
docker logs my-container                  # all logs
docker logs --tail 100 my-container       # last 100 lines
docker logs --since 5m my-container       # last 5 minutes
docker logs -f my-container               # follow (live tail)
docker logs --timestamps my-container     # show timestamps

# Combine flags for precision
docker logs --since 2024-01-01T10:00:00 --until 2024-01-01T11:00:00 my-app
```

### Common Log Drivers

- `json-file` -- default, stored on disk
- `local` -- optimised, compressed
- `syslog` -- sends to syslog daemon
- `journald` -- systemd journal
- `fluentd` / `gelf` / `awslogs`

### Gotcha: Missing Logs

- App writes to file instead of stdout
- Non-default log driver hides `docker logs`
- Log rotation truncated old entries
- Container was recreated (logs lost)

```bash
# Check which log driver a container uses
docker inspect --format='{{.HostConfig.LogConfig.Type}}' my-container

# Limit log file size in daemon.json or per-container
docker run --log-opt max-size=10m --log-opt max-file=3 my-image
```

---

## docker exec for Live Debugging

Shell into a **running** container to inspect state, test connectivity, and debug in real time.

```bash
# Interactive shell
docker exec -it my-container /bin/bash
docker exec -it my-container /bin/sh         # alpine / minimal images

# Run a single command
docker exec my-container cat /etc/hosts
docker exec my-container env                 # check environment variables
docker exec my-container ps aux              # see running processes

# As a specific user
docker exec -u root my-container apt-get update

# Set environment variables for the exec session
docker exec -e DEBUG=true my-container ./run-diagnostics.sh
```

### Useful In-Container Checks

- `cat /etc/resolv.conf` -- DNS config
- `ping` / `curl` / `wget` -- connectivity
- `df -h` -- disk space inside container
- `top` / `htop` -- process resource usage
- `ls -la /app` -- verify files & permissions

### When exec Won't Work

- Container is stopped -- use `docker run` instead
- No shell in image (e.g., distroless/scratch)
- Container restarts too fast to attach
- **Workaround:** `docker run -it --entrypoint /bin/sh image`
- **Debug images:** `docker debug my-container` (Docker Desktop)

---

## docker inspect Deep Dive

`docker inspect` returns the complete JSON configuration of any Docker object -- the single most powerful diagnostic command.

```bash
# Full JSON output (pipe to jq for readability)
docker inspect my-container | jq .

# Targeted queries with Go templates
docker inspect -f '{{.State.Status}}' my-container
docker inspect -f '{{.NetworkSettings.IPAddress}}' my-container
docker inspect -f '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}' my-container
docker inspect -f '{{json .Config.Env}}' my-container | jq .

# Inspect images and networks too
docker image inspect nginx:latest -f '{{.Config.ExposedPorts}}'
docker network inspect bridge | jq '.[0].Containers'
```

### Key Sections

- **.State** -- Status, ExitCode, OOMKilled, StartedAt, FinishedAt, Pid
- **.NetworkSettings** -- IPAddress, Gateway, Port mappings, DNS settings
- **.Config & .Mounts** -- Env vars, Labels, Entrypoint & Cmd, Volume bindings

---

## docker stats & Resource Monitoring

Real-time resource usage for all running containers -- your first stop for performance issues.

```bash
# Live dashboard (updates every second)
docker stats

# One-shot snapshot (good for scripts)
docker stats --no-stream

# Specific containers with custom format
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}" web api db

# Check memory limits vs actual usage
docker inspect -f '{{.HostConfig.Memory}}' my-container   # 0 = unlimited
```

| Column | What It Shows | Warning Signs |
|--------|---------------|---------------|
| CPU % | CPU usage vs available cores | >100% = using multiple cores heavily |
| MEM USAGE/LIMIT | Current memory / hard limit | Approaching limit = OOM risk |
| NET I/O | Network bytes in/out | Unexpectedly high = possible loop |
| BLOCK I/O | Disk read/write bytes | High writes = logging or temp files |
| PIDs | Number of processes | Growing count = possible fork bomb |

### Setting Resource Limits

```bash
docker run --memory=512m --cpus=1.5 --pids-limit=100 my-image
```

Always set limits in production to prevent a single container from taking down the host.

---

## Debugging Build Failures

Build failures happen at **image creation time**. The key is identifying which layer failed and why.

### Common Build Errors

- **COPY failed:** file not in build context, check `.dockerignore`
- **RUN failed:** command returned non-zero, missing dependency
- **Package not found:** need `apt-get update` before `install`
- **Permission denied:** wrong file permissions on scripts
- **Network errors:** DNS resolution during build

### Debugging Techniques

- Use `--progress=plain` for full output
- Use `--no-cache` to rule out stale layers
- Add `RUN ls -la` to verify file state
- Build up to a specific stage with `--target`
- Use `DOCKER_BUILDKIT=0` to get intermediate IDs

```bash
# Verbose build output
docker build --progress=plain --no-cache -t myapp .

# Debug a failed multi-stage build at a specific stage
docker build --target=builder -t myapp-debug .
docker run -it myapp-debug /bin/sh

# With BuildKit, use inline cache for CI
docker build --build-arg BUILDKIT_INLINE_CACHE=1 -t myapp .
```

---

## Layer Caching Issues

Docker caches each layer. When the cache breaks unexpectedly -- or does not break when it should -- you get confusing behaviour.

### Cache Not Working (Slow Builds)

- Copying `.` before installing deps invalidates cache on every file change
- `ADD` with URLs always re-downloads
- Build arg changes invalidate all downstream layers
- Different build context (CI vs local)

### Stale Cache (Wrong Output)

- `apt-get update` cached with old package list
- Git clone caches a commit forever
- `pip install -r requirements.txt` cached but file changed
- Fix: always `--no-cache` for CI releases

```dockerfile
# BAD: cache busts on any source change
COPY . /app
RUN npm install

# GOOD: install deps first, then copy source
COPY package.json package-lock.json /app/
RUN npm install
COPY . /app
```

```bash
# Bust cache from a specific point using build arg
ARG CACHE_BUST=1
RUN git clone https://github.com/org/repo.git
```

---

## Networking Problems

Networking is the most common source of "works locally, fails in Docker" issues.

### DNS Failures

- Check `/etc/resolv.conf`
- Custom DNS: `--dns 8.8.8.8`
- Service discovery needs user-defined network
- Default bridge has no auto DNS

### Connectivity

- Container-to-container: use network name
- `localhost` inside container != host
- Use `host.docker.internal` for host
- Firewall rules blocking traffic

### Port Conflicts

- `port is already allocated`
- `lsof -i :8080` to find culprit
- App listening on 127.0.0.1, not 0.0.0.0
- EXPOSE != publish; need `-p`

```bash
# Debug networking step by step
docker network ls                                        # list networks
docker network inspect my-network                        # show connected containers
docker exec my-container cat /etc/resolv.conf            # check DNS config
docker exec my-container ping other-container             # test connectivity
docker exec my-container curl -v http://api:3000/health  # test service discovery

# Create a user-defined network for DNS resolution
docker network create my-app-net
docker run --network my-app-net --name api my-api-image
docker run --network my-app-net --name web my-web-image  # can reach "api" by name
```

---

## Volume & Permission Issues

Bind mounts and volumes cause some of the most frustrating debugging sessions, especially around **file permissions**.

### Common Volume Problems

- Empty directory after mount (overrides image files)
- Changes not appearing (cached/wrong path)
- Bind mount shows host user UID inside container
- SELinux blocking access (need `:z` or `:Z`)
- Windows line endings breaking scripts (CRLF)

### Permission Fixes

- Match container UID to host UID
- `RUN chown -R appuser:appuser /app`
- `docker run --user $(id -u):$(id -g)`
- Use named volumes instead of bind mounts
- Set `umask` in entrypoint script

```bash
# Debug volume/permission issues
docker exec my-container ls -la /app          # check ownership
docker exec my-container id                    # who am I in the container?
docker inspect -f '{{json .Mounts}}' my-container | jq .  # list mounts

# Fix ownership at runtime
docker exec -u root my-container chown -R 1000:1000 /data

# SELinux context (Fedora/RHEL)
docker run -v /host/data:/data:z my-image     # shared label
docker run -v /host/data:/data:Z my-image     # private label
```

---

## Container Keeps Restarting

A container in a restart loop is one of the most common and frustrating patterns. The key is breaking the cycle to inspect state.

### Why Containers Restart

- App crashes immediately on startup
- Missing environment variable or config file
- Database not ready (race condition)
- Port already in use inside container
- OOM killed repeatedly (memory limit too low)
- Health check failing, causing orchestrator restart

### How to Break the Loop

- Check logs: `docker logs --tail 50 my-app`
- Stop the restart: `docker update --restart=no my-app`
- Override entrypoint: `docker run --entrypoint /bin/sh -it image`
- Check restart count: `docker inspect -f '{{.RestartCount}}'`
- Check OOM: `docker inspect -f '{{.State.OOMKilled}}'`

```bash
# Full restart debugging workflow
docker ps -a | grep my-app                           # see status & restart count
docker logs --tail 50 my-app                         # read the crash output
docker inspect -f '{{.State.ExitCode}}' my-app       # what exit code?
docker inspect -f '{{.State.OOMKilled}}' my-app      # was it OOM?
docker update --restart=no my-app                    # stop the loop
docker run -it --entrypoint /bin/sh my-image         # debug interactively
```

---

## Image Pull Errors & Registry Issues

### Common Pull Errors

| Error | Cause |
|-------|-------|
| `manifest unknown` | Tag does not exist in registry |
| `pull access denied` | Private repo, need `docker login` |
| `no matching manifest for linux/arm64` | Image not built for your architecture |
| `TLS handshake timeout` | Network issue or proxy config |
| `toomanyrequests` | Docker Hub rate limit (100 pulls/6h) |

### Solutions

- Verify tag exists: check registry UI or API
- Login: `docker login registry.example.com`
- Check arch: `docker manifest inspect image:tag`
- Use mirror/proxy for rate limits
- Pin digests for reproducibility: `image@sha256:abc...`
- Insecure registry: add to `daemon.json`

```bash
# Debug pull issues
docker pull --platform linux/amd64 nginx:latest      # force platform
docker manifest inspect nginx:latest                  # check available architectures
docker login ghcr.io                                  # authenticate to GitHub registry
```

---

## Disk Space Problems

Docker can silently consume **tens of gigabytes** with old images, stopped containers, and unused volumes.

```bash
# See what Docker is using
docker system df                     # summary: images, containers, volumes, build cache
docker system df -v                  # detailed breakdown per object

# The nuclear option: remove everything unused
docker system prune -a --volumes     # WARNING: removes all stopped containers,
                                     # unused images, unused networks, and volumes
```

### Targeted Cleanup

- `docker container prune` -- remove stopped containers
- `docker image prune -a` -- remove unused images
- `docker volume prune` -- remove orphan volumes
- `docker builder prune` -- clear build cache
- `docker image prune --filter "until=720h"` -- older than 30 days

### Prevention

- Use `--rm` flag for ephemeral containers
- Set up cron job: `docker system prune -f`
- Use multi-stage builds (smaller images)
- Add `.dockerignore` to exclude large files
- Use Alpine or distroless base images

```bash
# Find the biggest images
docker images --format "{{.Size}}\t{{.Repository}}:{{.Tag}}" | sort -hr | head -10

# Find containers writing lots of data to their writable layer
docker ps -s --format "table {{.Names}}\t{{.Size}}"
```

---

## Performance Debugging

When containers are slow, systematically check **CPU**, **memory**, **I/O**, and **networking**.

### CPU Issues

- `docker stats` shows CPU %
- Check CPU limits: `--cpus`, `--cpu-shares`
- `docker exec my-app top -bn1` -- find hot process
- Host-level: `htop`, check steal time in VMs

### Memory Issues

- `docker stats` shows MEM USAGE / LIMIT
- OOM kills: `dmesg | grep -i oom`
- Java apps: set `-XX:MaxRAMPercentage=75`
- Node.js: `--max-old-space-size=384`

### I/O Bottlenecks

- Overlay2 writes are slower than native
- Use volumes for write-heavy workloads
- `docker exec my-app iostat` (if available)
- tmpfs mounts for temp data: `--tmpfs /tmp`

### Network Latency

- Bridge NAT adds ~0.5ms overhead
- `--network host` removes NAT (Linux only)
- Check MTU mismatches across networks
- DNS resolution caching in container

```bash
# Host-level process inspection
docker inspect -f '{{.State.Pid}}' my-container     # get PID on host
sudo strace -p <PID> -c                             # syscall profiling
sudo nsenter -t <PID> -n ss -tlnp                   # network sockets from container namespace
```

---

## Docker Daemon Logs

When the problem is not the container but **Docker itself** -- daemon logs reveal engine-level errors.

### Finding Daemon Logs

| OS / Init | Command |
|-----------|---------|
| systemd (most Linux) | `journalctl -u docker.service -f` |
| macOS (Docker Desktop) | `~/Library/Containers/.../log/vm/dockerd.log` |
| Windows (Docker Desktop) | Event Viewer -> Application |
| Amazon Linux / CentOS | `/var/log/messages` |

### Enable Debug Mode

- Add `"debug": true` to `/etc/docker/daemon.json`
- Reload: `sudo systemctl reload docker`
- Or send signal: `sudo kill -SIGHUP $(pidof dockerd)`
- Produces verbose output for all operations

```bash
# Check daemon status
sudo systemctl status docker
sudo journalctl -u docker.service --since "1 hour ago"

# Enable debug logging
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "debug": true,
  "log-level": "debug"
}
EOF
sudo systemctl reload docker
```

### Common Daemon Issues

Storage driver errors, out of disk space, failed to start (port 2376 in use), DNS resolution failures, cgroup v1/v2 incompatibility, AppArmor/SELinux denials.

---

## Useful Debugging Tools

Third-party tools that make Docker troubleshooting significantly easier.

### dive -- Image Layer Explorer

- Visualise each layer and its file changes
- Find wasted space and bloated layers
- `dive my-image:latest`
- CI mode: `dive --ci my-image` (fails on waste)

### ctop -- Container Top

- Real-time metrics for all containers
- Like `htop` but for Docker
- CPU, memory, network, I/O at a glance
- `docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock quay.io/vektorlab/ctop`

### lazydocker -- Terminal UI

- Full Docker management in the terminal
- Logs, stats, inspect in one view
- Compose-aware, shows service groups
- `brew install lazydocker` or `go install`

### nsenter -- Namespace Entry

- Enter container namespaces from the host
- Works even without shell in the container
- Debug networking: `nsenter -t PID -n ip addr`
- Debug mounts: `nsenter -t PID -m cat /etc/hosts`

```bash
# Install and run common tools
brew install dive lazydocker ctop       # macOS
apt install -y strace ltrace tcpdump    # Linux debugging essentials

# Use docker debug (Docker Desktop 4.27+)
docker debug my-container               # drops into a shell with tools pre-installed
```

---

## Troubleshooting Cheat Sheet

| Symptom | First Command | Next Step |
|---------|---------------|-----------|
| Container won't start | `docker logs <name>` | Check exit code with `docker inspect` |
| Container keeps restarting | `docker logs --tail 50` | `docker update --restart=no` |
| App not reachable | `docker port <name>` | Check app binds to `0.0.0.0` |
| Container slow | `docker stats` | Check limits, profile with `strace` |
| Out of disk space | `docker system df` | `docker system prune -a` |
| Build failing | `docker build --progress=plain` | `--no-cache`, check `.dockerignore` |
| Can't pull image | `docker manifest inspect` | `docker login`, check tag exists |
| Permission denied | `docker exec ls -la /app` | Match UID/GID, check SELinux |
| DNS not resolving | `docker exec cat /etc/resolv.conf` | Use user-defined network |
| OOM killed | `docker inspect {{.State.OOMKilled}}` | Increase `--memory` limit |

### Universal Debug One-Liner

```bash
docker ps -a && docker logs --tail 30 <container> && docker inspect -f '{{json .State}}' <container> | jq .
```

---

## Summary & Further Reading

### Key Takeaways

- Always start with `docker logs` and `docker inspect`
- Exit codes tell you *how* the container died
- Use user-defined networks for DNS resolution
- Set resource limits in production
- Regularly prune unused Docker objects
- Know your tools: dive, ctop, lazydocker, nsenter

### Debugging Mindset

- Reproduce before you fix
- Change one thing at a time
- Check the simple things first
- Read the error message carefully
- Document what you find for the team

### Official Documentation

- [Docker Daemon Logs](https://docs.docker.com/engine/daemon/logs/)
- [docker logs Reference](https://docs.docker.com/reference/cli/docker/container/logs/)
- [docker inspect Reference](https://docs.docker.com/reference/cli/docker/inspect/)
- [Docker Networking Guide](https://docs.docker.com/engine/network/)
- [Docker Storage Guide](https://docs.docker.com/storage/)

### Tools & Resources

- [dive](https://github.com/wagoodman/dive) -- image layer analysis
- [ctop](https://github.com/bcicen/ctop) -- container metrics
- [lazydocker](https://github.com/jesseduffield/lazydocker) -- terminal UI
- [docker-bench-security](https://github.com/docker/docker-bench-security) -- security audit
- [netshoot](https://github.com/nicolaka/netshoot) -- network troubleshooting
