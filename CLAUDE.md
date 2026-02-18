# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Helm chart repository that deploys Docker-in-Docker (dind) as a Kubernetes service. The chart is published via GitHub Pages using `helm/chart-releaser-action` and hosted at `https://peterwwillis.github.io/helm-docker-in-docker/`.

## Common Commands

```bash
# Lint the chart
helm lint charts/docker-in-docker

# Render templates locally (dry-run, useful for previewing changes)
helm template test-release charts/docker-in-docker

# Render with specific values overrides
helm template test-release charts/docker-in-docker --set secureDocker=false

# Package the chart manually
helm package charts/docker-in-docker

# Validate Kubernetes manifests (requires kubectl)
helm template test-release charts/docker-in-docker | kubectl apply --dry-run=client -f -

# Install from local chart
helm install dind charts/docker-in-docker --generate-name
```

## Architecture

The chart deploys two main components in a single Pod:

1. **dind container** (`docker.io/docker:24.0.2-dind`) — the Docker daemon. When `secureDocker: true` (default), it auto-generates TLS certs and listens on port 2376. When false, listens insecurely on 2375.

2. **gc container** (`drone/gc`) — the Drone Garbage Collector sidecar that periodically cleans up unused Docker images/containers. Enabled by default.

### TLS / Secrets (`templates/secrets.yaml`)

When `secureDocker: true`, Helm's `genCA`/`genSignedCert` functions generate two Kubernetes TLS Secrets at install time:
- `<release>-cert-server` — mounted into the dind container
- `<release>-cert-client` — mounted into the dind and gc containers; also downloaded by the Docker client to authenticate

### Bridge / Multus Networking (`templates/bridge-daemonset.yaml`, `templates/network-attachment-definition.yaml`)

When `bridge.enabled: true`, the chart creates additional resources for multi-interface NAT routing via Multus CNI:
- **NetworkAttachmentDefinition** — a Multus bridge network (`net1` interface) attached to the dind Pod with a static IP (`bridge.staticIP`)
- **DaemonSet** (`<release>-bridge-nat`) — runs a privileged Alpine container on every node that sets up iptables rules (DNAT, MASQUERADE) and optionally policy-based routing (separate routing table + `ip rule fwmark`) so the bridge subnet is reachable from any host interface

The bridge DaemonSet uses a ConfigMap containing a `setup-nat.sh` script that is fully rendered by Helm from `values.yaml` settings (subnet, port forwarding rules, routing table, fwmark, outbound interface).

When `bridge.enabled: true`, the Service type is forced to `ClusterIP` (ignoring `service.type`).

### Key `values.yaml` Toggles

| Value | Effect |
|---|---|
| `secureDocker` | TLS on/off; changes port 2376↔2375 and mounts cert secrets |
| `bridge.enabled` | Enables Multus NetworkAttachmentDefinition + NAT DaemonSet; forces Service to ClusterIP |
| `bridge.routing.enabled` | Adds policy-based routing (custom routing table + fwmark) to the DaemonSet |
| `gc.enabled` | Enables/disables the Drone GC sidecar |
| `service.extraPorts` | Additional ports exposed on the Service and dind container |
| `bridge.portForwarding.rules` | DNAT rules injected into the DaemonSet's iptables setup |

## CI/CD

The GitHub Actions workflow (`.github/workflows/branch-main.yml`) triggers on pushes to `main` and:
1. Runs `helm/chart-releaser-action` to package and publish the chart (creates GitHub Releases with `.tgz` artifacts and updates `gh-pages:index.yaml`)
2. Generates HTML docs from `charts/docker-in-docker/README.md` via `ldeluigi/markdown-docs`
3. Deploys HTML + `index.yaml` to the `gh-pages` branch

**To release a new chart version**: bump `version` in `charts/docker-in-docker/Chart.yaml` and push to `main`. The CI will skip existing versions (`CR_SKIP_EXISTING=true`).
