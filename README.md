# zigbee2mqtt-kustomize

Kubernetes manifests for deploying [Zigbee2MQTT](https://www.zigbee2mqtt.io/) using [Kustomize](https://kustomize.io/), with first-class support for [FluxCD](https://fluxcd.io/) GitOps workflows.

## What is Zigbee2MQTT?

[Zigbee2MQTT](https://www.zigbee2mqtt.io/) is a bridge that exposes your Zigbee network to MQTT, enabling integration with home automation systems. It connects a USB or serial Zigbee coordinator (such as the Texas Instruments CC2652 series or Sonoff Zigbee 3.0 USB Dongle Plus) to an MQTT broker, and publishes device state and events as MQTT messages.

## How It Integrates With Home Assistant

When `homeassistant.enabled: true` is set in the Zigbee2MQTT configuration, Zigbee2MQTT publishes [MQTT Discovery](https://www.home-assistant.io/integrations/mqtt/#mqtt-discovery) messages to your Home Assistant MQTT broker. Home Assistant automatically creates entities for all paired Zigbee devices — no manual entity configuration required.

**Prerequisites on the Home Assistant side:**
- The [MQTT integration](https://www.home-assistant.io/integrations/mqtt/) must be configured in Home Assistant.
- Home Assistant and Zigbee2MQTT must point to the same MQTT broker.

## Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│  Kubernetes Cluster                                       │
│                                                           │
│  ┌─────────────────────────┐                             │
│  │  zigbee2mqtt StatefulSet│                             │
│  │                         │                             │
│  │  /dev/ttyACM0 (USB)─────┼──► Zigbee Coordinator     │
│  │  /app/data (hostPath/   │                             │
│  │            PVC)         │                             │
│  │  Port 8080 (web UI)     │                             │
│  └────────────┬────────────┘                             │
│               │                                           │
│               │ MQTT (port 1883)                          │
│               ▼                                           │
│  ┌────────────────────────┐    ┌──────────────────────┐  │
│  │   MQTT Broker          │◄──►│  Home Assistant       │  │
│  │   (Mosquitto etc.)     │    │  (MQTT Integration)   │  │
│  └────────────────────────┘    └──────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

## Key Features

- **StatefulSet deployment** — ensures stable network identity and reliable restarts.
- **Kustomize-native** — apply directly or use as a Kustomize remote base.
- **FluxCD ready** — works as a GitRepository source for FluxCD Kustomization resources.
- **Renovate enabled** — automatic image version PRs via [Renovate](https://docs.renovatebot.com/).
- **Configurable** — optional ConfigMap for GitOps-managed configuration, or bring your own data directory.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Kubernetes cluster | k3s, k0s, RKE2, EKS, GKE, etc. |
| Zigbee USB coordinator | Connected to a cluster node (e.g. `/dev/ttyACM0`) |
| MQTT broker | Mosquitto or similar, reachable from the cluster |
| `kubectl` | For direct install |
| `kustomize` | v5+ (or `kubectl apply -k`) |
| FluxCD (optional) | For GitOps install |

> **Node affinity:** Because the Zigbee USB device is physically attached to a specific node, you will likely need to pin the pod to that node using a `nodeSelector` or `nodeName`. See [Configuration](#configuration) for details.

---

## Installation

### Method 1 — kubectl (Direct Install)

**Step 1: Clone or download this repository**

```bash
git clone https://github.com/turbo5000c/zigbee2mqtt-kustomize.git
cd zigbee2mqtt-kustomize
```

**Step 2: Edit the StatefulSet to match your environment**

Open `base/statefulset.yaml` and update the following values:

| Field | Default | Description |
|---|---|---|
| `spec.template.spec.volumes[zoneinfo].hostPath.path` | `/usr/share/zoneinfo` | Path to zoneinfo on the node |
| `spec.template.spec.volumes[zigbee2mqtt-data].hostPath.path` | `/var/lib/zigbee2mqtt/data` | **Required:** path on the node where Zigbee2MQTT data is stored |
| `spec.template.spec.containers[0].env[TZ]` | `UTC` | Timezone (e.g. `America/New_York`) |
| `spec.template.spec.containers[0].volumeMounts[zoneinfo].subPath` | `America/New_York` | Must match the `TZ` value |

**Step 3: Edit the ConfigMap (optional)**

Uncomment the `configmap.yaml` entry in `base/kustomization.yaml` to manage the Zigbee2MQTT `configuration.yaml` via a ConfigMap. Then update the MQTT server address in `base/configmap.yaml`:

```yaml
mqtt:
  server: mqtt://<your-mqtt-broker-ip>:1883
```

Also update the serial port if your coordinator is not on `/dev/ttyACM0`.

**Step 4: Create the namespace and deploy**

```bash
kubectl create namespace zigbee2mqtt
kubectl apply -k . -n zigbee2mqtt
```

**Step 5: Verify the deployment**

```bash
kubectl get pods -n zigbee2mqtt
kubectl logs -n zigbee2mqtt -l app=zigbee2mqtt
```

The Zigbee2MQTT web UI is available at `http://<node-ip>:8080` (or via a port-forward):

```bash
kubectl port-forward -n zigbee2mqtt svc/zigbee2mqtt 8080:8080
```

---

### Method 2 — FluxCD (GitOps)

FluxCD users can reference this repository directly as a GitRepository source and apply it as a Kustomization.

**Step 1: Create the namespace and any required Secrets**

```bash
kubectl create namespace zigbee2mqtt
```

**Step 2: Apply the FluxCD resources**

Copy the example files from [`examples/flux/`](examples/flux/) and update them to match your cluster. Then apply:

```bash
kubectl apply -f examples/flux/gitrepository.yaml
kubectl apply -f examples/flux/kustomization.yaml
```

Or place these files in your main Flux repository so Flux manages them automatically.

**Example GitRepository:**

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: zigbee2mqtt-kustomize
  namespace: flux-system
spec:
  interval: 1h
  url: https://github.com/turbo5000c/zigbee2mqtt-kustomize
  ref:
    branch: main
```

**Example Kustomization:**

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: zigbee2mqtt
  namespace: flux-system
spec:
  interval: 10m
  path: "./"
  prune: true
  sourceRef:
    kind: GitRepository
    name: zigbee2mqtt-kustomize
  targetNamespace: zigbee2mqtt
  postBuild:
    substitute:
      DATA_PATH: "/your/data/path"
      TZ: "America/New_York"
```

See [`examples/flux/`](examples/flux/) for complete, ready-to-use example files.

---

## Configuration

### Timezone

The timezone is set in two places in `statefulset.yaml`:

1. The `TZ` environment variable (used by the application):
   ```yaml
   env:
     - name: TZ
       value: America/New_York
   ```

2. The `zoneinfo` volume mount `subPath` (provides the correct localtime file):
   ```yaml
   volumeMounts:
     - name: zoneinfo
       mountPath: /etc/localtime
       subPath: America/New_York
   ```

Both values must match.

### Serial Port / Zigbee Coordinator

Edit `base/configmap.yaml` to set your coordinator's serial port:

```yaml
serial:
  port: /dev/ttyACM0   # Change to match your device
  baudrate: 115200
  rtscts: false
```

Common port values:
- `/dev/ttyACM0` — CP210x or CH340 based coordinators
- `/dev/ttyUSB0` — Some coordinators enumerate here
- `/dev/serial/by-id/<id>` — Recommended for stability (survives reboots)

> The pod runs with `privileged: true` and `SYS_ADMIN` capability to access the USB serial device. This is required for direct device access.

### MQTT Broker

Edit the `mqtt.server` value in `base/configmap.yaml`:

```yaml
mqtt:
  server: mqtt://mosquitto.home-assistant.svc.cluster.local:1883
  # Optional: add authentication
  # user: zigbee2mqtt
  # password: yourpassword
```

If your MQTT broker requires authentication, add `user` and `password` fields. Consider using a Kubernetes Secret and injecting the values via environment variables or a Kustomize secret generator.

### Node Pinning

Because the Zigbee USB dongle is physically attached to one node, pin the pod to that node. Add a `nodeSelector` to the StatefulSet spec:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/hostname: my-node-name
```

Or use `nodeName` for a hard pin:

```yaml
spec:
  template:
    spec:
      nodeName: my-node-name
```

### Data Storage

By default, the StatefulSet mounts a `hostPath` volume for Zigbee2MQTT's data directory (`/app/data`). This stores the device database, coordinator backup, and configuration state.

**Using hostPath (default):**

Update the path in `base/statefulset.yaml` to point to an existing directory on your node:

```yaml
volumes:
  - name: zigbee2mqtt-data
    hostPath:
      path: /your/data/path   # Must exist on the node
```

**Using a PersistentVolumeClaim:**

See [`examples/pvc/`](examples/pvc/) for a Kustomize overlay that replaces the hostPath with a PVC. This is recommended when using a distributed storage provider (Longhorn, NFS, etc.).

### Configuration File (ConfigMap)

The `configmap.yaml` contains a full `configuration.yaml` for Zigbee2MQTT. It is commented out of `kustomization.yaml` by default to allow the configuration to be managed by Zigbee2MQTT itself (stored in the data directory).

To manage the configuration via a ConfigMap (GitOps style):

1. Uncomment the ConfigMap in `base/kustomization.yaml`:
   ```yaml
   resources:
     - statefulset.yaml
     - service.yaml
     - configmap.yaml   # Uncomment this line
   ```

2. Also uncomment the `config-volume` entries in `base/statefulset.yaml` to mount the ConfigMap.

3. Edit `configmap.yaml` with your settings.

> **Note:** Some Zigbee2MQTT settings (like the coordinator backup and device pairing state) are always stored in the data directory and cannot be fully managed via ConfigMap.

---

## Upgrading

### Automatic (Renovate)

This repository includes a `renovate.json` configuration. When used with [Renovate](https://docs.renovatebot.com/), Renovate automatically opens pull requests to update the `koenkk/zigbee2mqtt` image version in `statefulset.yaml`.

To enable this:
1. Install the [Renovate GitHub App](https://github.com/apps/renovate) on your fork.
2. Renovate will open PRs when new versions are available.
3. Merge the PR — FluxCD (or your next `kubectl apply`) will roll out the update.

### Manual

Update the image tag in `statefulset.yaml`:

```yaml
image: koenkk/zigbee2mqtt:2.9.1   # Change to the desired version
```

Check the [Zigbee2MQTT releases page](https://github.com/Koenkk/zigbee2mqtt/releases) for available versions.

Then apply:

```bash
kubectl apply -k . -n zigbee2mqtt
# or with FluxCD: git commit and push the change
```

---

## Troubleshooting

### Pod fails to start / CrashLoopBackOff

```bash
kubectl describe pod -n zigbee2mqtt -l app=zigbee2mqtt
kubectl logs -n zigbee2mqtt -l app=zigbee2mqtt --previous
```

**Common causes:**

| Symptom | Cause | Fix |
|---|---|---|
| `Error: failed to open serial port` | Wrong serial port or USB not attached to this node | Check port in configmap.yaml; pin pod to the correct node |
| `ENOENT /app/data` | Data directory does not exist | Create the hostPath directory on the node before deploying |
| `Connection refused` to MQTT | Wrong MQTT server address | Update `mqtt.server` in configmap.yaml |
| Pod keeps restarting | Liveness probe fails | Check pod logs; ensure port 8080 is reachable |

### Checking Serial Devices on a Node

```bash
# SSH to the node and list serial devices
ls -la /dev/tty*
ls -la /dev/serial/by-id/
```

### Accessing the Web UI

```bash
kubectl port-forward -n zigbee2mqtt svc/zigbee2mqtt 8080:8080
```

Then open `http://localhost:8080` in your browser.

### Resetting the Coordinator

To factory-reset the coordinator (removes all paired devices), use the Zigbee2MQTT web UI:
**Settings → Tools → Reset coordinator**

---

## Repository Structure

```
.
├── README.md                   # This file
├── kustomization.yaml          # Root Kustomize entry point (references base/)
├── renovate.json               # Renovate image auto-update config
├── base/                       # Base Kubernetes manifests
│   ├── kustomization.yaml      # Base Kustomize configuration
│   ├── statefulset.yaml        # Zigbee2MQTT StatefulSet
│   ├── service.yaml            # ClusterIP Service (port 8080)
│   └── configmap.yaml          # Optional: Zigbee2MQTT configuration.yaml
└── examples/
    ├── flux/                   # FluxCD GitRepository + Kustomization examples
    │   ├── README.md
    │   ├── gitrepository.yaml
    │   └── kustomization.yaml
    └── pvc/                    # Kustomize overlay: PVC-based storage
        ├── kustomization.yaml
        └── pvc.yaml
```

---

## Contributing

Issues and pull requests are welcome. Please ensure any manifest changes are tested against a real cluster before submitting.

## License

See [LICENSE](LICENSE) if present, or check the repository settings.
