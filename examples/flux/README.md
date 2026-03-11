# FluxCD Installation Examples

This directory contains example FluxCD resources for deploying Zigbee2MQTT.

## Files

| File | Description |
|---|---|
| `gitrepository.yaml` | FluxCD `GitRepository` — tracks this repository |
| `kustomization.yaml` | FluxCD `Kustomization` — applies manifests and patches your settings |

## Usage

### Step 1: Apply the GitRepository

```bash
kubectl apply -f gitrepository.yaml -n flux-system
```

Verify it is ready:

```bash
kubectl get gitrepository -n flux-system zigbee2mqtt-kustomize
```

### Step 2: Edit the Kustomization

Open `kustomization.yaml` and update:

- `spec.patches[0].patch.spec.template.spec.nodeName` — set to the node with your USB dongle.
- `spec.patches[0].patch.spec.template.spec.containers[0].env[TZ].value` — your timezone.
- `spec.patches[0].patch.spec.template.spec.containers[0].volumeMounts[zoneinfo].subPath` — same timezone.
- `spec.patches[0].patch.spec.template.spec.volumes[zigbee2mqtt-data].hostPath.path` — your data directory.

### Step 3: Create the namespace and apply

```bash
kubectl create namespace zigbee2mqtt
kubectl apply -f kustomization.yaml -n flux-system
```

### Step 4: Watch reconciliation

```bash
flux get kustomization zigbee2mqtt -n flux-system
kubectl get pods -n zigbee2mqtt
```

## Using This in Your Flux Repository

Place both files inside your Flux repository (e.g. `clusters/my-cluster/zigbee2mqtt/`).
Flux will pick them up automatically on its next reconciliation cycle.

```
clusters/
└── my-cluster/
    └── zigbee2mqtt/
        ├── gitrepository.yaml
        └── kustomization.yaml
```
