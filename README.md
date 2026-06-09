# gitops-platform

GitOps repository for platform operator lifecycle management using the Argo CD app-of-apps pattern on OpenShift.

## Repository Structure

```
gitops-platform/
├── clusters/
│   ├── hub/
│   │   ├── preproduction/
│   │   │   ├── applications/
│   │   │   │   ├── external-secrets-operator.yaml
│   │   │   │   ├── openshift-gitops-operator.yaml
│   │   │   │   └── kustomization.yaml
│   │   │   ├── external-secrets-operator/
│   │   │   │   └── kustomization.yaml
│   │   │   ├── openshift-gitops-operator/
│   │   │   │   └── kustomization.yaml
│   │   │   └── app-of-apps.yaml
│   │   └── production/
│   └── spoke/
│       ├── development/
│       ├── integration/
│       └── production/
├── resources/
│   ├── external-secrets-operator/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── operator-group.yaml
│   │   └── subscription.yaml
│   └── openshift-gitops-operator/
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       ├── operator-group.yaml
│       └── subscription.yaml
├── bootstrap.yaml
├── requirements.txt
├── requirements.yaml
└── README.md
```

## Concepts

### `resources/`

Reusable Kustomize bases for each operator. Each base contains the Namespace, OperatorGroup, and Subscription needed to install an operator via OLM. These are environment-agnostic and should never contain environment-specific values.

### `clusters/`

Cluster-specific configuration organised by role and environment:

```
clusters/<role>/<environment>/
```

| Role | Description |
|---|---|
| `hub` | ACM hub cluster |
| `spoke` | Managed spoke clusters |

Each environment directory contains:

| Path | Purpose |
|---|---|
| `app-of-apps.yaml` | Argo CD Application that syncs the `applications/` directory |
| `applications/` | Argo CD Application manifests for each operator, with sync-wave ordering |
| `<operator>/` | Kustomize overlay referencing the base in `resources/` — add patches here for environment-specific customisation |

### `bootstrap.yaml`

Ansible playbook to bootstrap a fresh cluster. Verifies cluster connectivity and applies the initial configuration.

## App-of-Apps Flow

```
bootstrap.yaml (Ansible)
  │
  └─▶ applies app-of-apps.yaml
        │
        └─▶ syncs clusters/hub/<env>/applications/
              │
              ├─▶ openshift-gitops-operator (sync-wave: 0)
              │     Source: clusters/hub/<env>/openshift-gitops-operator/
              │       └─▶ kustomize base: resources/openshift-gitops-operator/
              │
              └─▶ external-secrets-operator (sync-wave: 1)
                    Source: clusters/hub/<env>/external-secrets-operator/
                      └─▶ kustomize base: resources/external-secrets-operator/
```

## Environment-Specific Patching

Overlays live alongside the applications they serve, inside the cluster environment directory. Each `<operator>/kustomization.yaml` references the shared base in `resources/` and can layer on environment-specific patches.

To override a value for a single environment (e.g. a different subscription channel in production), add a patch file to the operator's overlay directory:

```
clusters/hub/production/external-secrets-operator/
├── kustomization.yaml
└── subscription-patch.yaml
```

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../../../resources/external-secrets-operator

patches:
  - path: subscription-patch.yaml
```

```yaml
# subscription-patch.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: external-secrets-operator
spec:
  channel: stable
```

## Prerequisites

- Python 3
- `oc` CLI authenticated to the target cluster
- Ansible Galaxy collection `kubernetes.core`

## Getting Started

```bash
# Create and activate the virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Install Python dependencies
pip install -r requirements.txt

# Install the required Ansible collections
ansible-galaxy collection install -r requirements.yaml

# Log in to the target cluster
oc login --server=<cluster-api-url>

# Bootstrap the cluster
ansible-playbook bootstrap.yaml
```
