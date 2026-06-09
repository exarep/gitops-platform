# gitops-platform

GitOps repository for platform operator lifecycle management using the Argo CD app-of-apps pattern.

## Repository Structure

```
gitops-platform/
├── clusters/
│   ├── hub/
│   │   ├── preproduction/
│   │   │   ├── applications/
│   │   │   │   ├── external-secrets-operator.yaml   # sync-wave: 1
│   │   │   │   ├── openshift-gitops-operator.yaml   # sync-wave: 0
│   │   │   │   └── kustomization.yaml
│   │   │   └── app-of-apps.yaml
│   │   └── production/
│   │       └── (planned)
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
└── README.md
```

## Layout

| Directory | Purpose |
|---|---|
| `clusters/` | Cluster-specific Argo CD Application definitions, organised by role and environment |
| `clusters/hub/` | Hub (ACM) cluster environments |
| `clusters/spoke/` | Managed spoke cluster environments |
| `resources/` | Reusable Kustomize bases for operator installations (Namespace, OperatorGroup, Subscription) |
| `bootstrap.yaml` | Ansible playbook to bootstrap a fresh cluster |

## App-of-Apps Flow

```
bootstrap.yaml (Ansible)
  └─▶ Installs operators, applies app-of-apps
        └─▶ clusters/hub/preproduction/
              └─▶ app-of-apps.yaml
                    └─▶ clusters/hub/preproduction/applications/
                          ├─▶ openshift-gitops-operator  (sync-wave: 0)
                          │     └─▶ resources/openshift-gitops-operator/
                          └─▶ external-secrets-operator   (sync-wave: 1)
                                └─▶ resources/external-secrets-operator/
```

## Getting Started

```bash
# Bootstrap a fresh cluster (requires oc login first)
ansible-playbook bootstrap.yaml
```