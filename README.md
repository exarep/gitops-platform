# gitops-platform

GitOps repository for platform operator lifecycle management using the Argo CD app-of-apps pattern on OpenShift.

## Prerequisites

- Python 3
- `oc` CLI authenticated to the target cluster
- AWS Secrets Manager secrets provisioned by the `iac` project (`pb-setup-secrets.yaml`)
- The `exarep/aws-credentials` secret populated with valid AWS credentials (JSON)
- The `exarep/test` secret populated for External Secrets Operator validation

## Getting started

```shell
git clone https://github.com/exarep/gitops-platform.git
cd gitops-platform
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yaml
```

## Bootstrap

Log in to the target cluster and run the bootstrap playbook:

```shell
oc login --server=https://api.hub.cluster.exarep.com:6443
ansible-playbook bootstrap.yaml
```

### What the bootstrap does

1. Verifies the `oc` CLI is authenticated to the cluster
2. Installs the External Secrets Operator and waits for the CSV to succeed
3. Applies the `ExternalSecretsConfig` and waits for the webhook deployment
4. Creates the `aws-secrets-manager-credentials` Kubernetes secret from local AWS credentials
5. Applies the `ClusterSecretStore` pointing to AWS Secrets Manager
6. Applies a test `ExternalSecret` and verifies the secret syncs successfully
7. *(Planned)* Installs the OpenShift GitOps Operator
8. *(Planned)* Creates the `gitops-platform` and `gitops-workloads` Argo CD instances
9. *(Planned)* Applies the app-of-apps to begin managing platform resources

## Project structure

```
gitops-platform/
├── ansible.cfg
├── bootstrap.yaml
├── requirements.txt
├── requirements.yaml
├── clusters/
│   └── hub/
├── resources/
│   ├── external-secrets-operator/
│   │   ├── namespace.yaml
│   │   ├── operator-group.yaml
│   │   ├── subscription.yaml
│   │   ├── external-secrets-config.yaml
│   │   ├── cluster-secret-store.yaml
│   │   └── test-external-secret.yaml
│   ├── openshift-gitops-operator/
│   │   ├── namespace.yaml
│   │   ├── operator-group.yaml
│   │   └── subscription.yaml
│   ├── gitops-platform/
│   │   ├── namespace.yaml
│   │   └── argocd.yaml
│   └── gitops-workloads/
│       ├── namespace.yaml
│       └── argocd.yaml
```

## Secrets integration

The bootstrap connects the cluster to AWS Secrets Manager via the External Secrets Operator. The flow is:

1. `iac/pb-setup-secrets.yaml` creates secrets in AWS Secrets Manager under the `exarep/` prefix
2. The bootstrap creates a `ClusterSecretStore` that authenticates to AWS using the `aws-secrets-manager-credentials` Kubernetes secret
3. `ExternalSecret` resources reference the `ClusterSecretStore` to sync AWS secrets into the cluster as Kubernetes secrets
4. A test `ExternalSecret` pulls `exarep/test` and verifies the full pipeline works end-to-end
