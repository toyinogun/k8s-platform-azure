# WordPress on Kubernetes (GitOps with ArgoCD)

This repository contains the GitOps configuration for deploying a WordPress site on Kubernetes using ArgoCD, Longhorn for storage, and Sealed Secrets for credential management. It is designed to run behind a Cloudflare Tunnel.

## Architecture

*   **Application**: WordPress (Bitnami image) & MariaDB (Bitnami image)
*   **Storage**: Longhorn (StorageClass: `longhorn`)
*   **Secrets**: Bitnami Sealed Secrets
*   **Networking**: Service exposed via Cloudflare Tunnel (no Ingress resource used directly)
*   **Deployment**: ArgoCD (GitOps)

## Prerequisites

Before deploying, ensure your cluster has the following components installed:

1.  **Kubernetes Cluster** (e.g., K3s, AKS, etc.)
2.  **ArgoCD** installed in the `argocd` namespace.
3.  **Longhorn** installed and configured as the storage provisioner.
4.  **Sealed Secrets Controller** installed in the `kube-system` namespace.
5.  **Cloudflare Tunnel** (`cloudflared`) deployment configured to route traffic to the `wordpress` service.

## Repository Structure

```
.
├── bootstrap/
│   └── wordpress-app.yaml  # ArgoCD Application manifest
├── wordpress/
│   ├── kustomization.yaml  # Kustomize entrypoint
│   ├── namespace.yaml      # Namespace definition
│   ├── wordpress.yaml      # WordPress Deployment & Service
│   ├── mysql.yaml          # MariaDB Deployment & Service
│   └── sealed-secret.yaml  # Encrypted secrets (safe for Git)
└── ...
```

## Setup & Deployment

### 1. Secret Generation (First Time Only)

If you need to update or reset passwords, you must generate a new `SealedSecret`.

1.  **Install `kubeseal`**:
    ```bash
    brew install kubeseal
    ```

2.  **Create a dry-run secret locally**:
    ```bash
    kubectl create secret generic wordpress-secrets \
      --from-literal=mariadb-root-password="YOUR_STRONG_PASSWORD" \
      --from-literal=mariadb-password="YOUR_STRONG_PASSWORD" \
      --from-literal=wordpress-password="YOUR_STRONG_PASSWORD" \
      --namespace wordpress \
      --dry-run=client -o yaml > secret.yaml
    ```

3.  **Seal the secret**:
    ```bash
    kubeseal --controller-name=sealed-secrets \
             --controller-namespace=kube-system \
             --format yaml < secret.yaml > wordpress/sealed-secret.yaml
    ```

4.  **Cleanup**: Delete the plain text `secret.yaml`.
    ```bash
    rm secret.yaml
    ```

5.  **Commit**: Push the new `wordpress/sealed-secret.yaml` to the repository.

### 2. Deployment via ArgoCD

1.  **Apply the Application Manifest**:
    ```bash
    kubectl apply -f bootstrap/wordpress-app.yaml
    ```

2.  **Sync**: ArgoCD will automatically detect the new application and sync the resources to the `wordpress` namespace.

### 3. Networking (Cloudflare Tunnel)

Ensure your Cloudflare Tunnel is configured to route traffic to the internal Kubernetes service:
*   **Service Name**: `wordpress.wordpress.svc.cluster.local`
*   **Port**: `80` (HTTP) or `443` (HTTPS)

*Note: The WordPress container listens on 8080/8443, but the Service exposes them on 80/443.*

## Verification

Check the status of the pods:

```bash
kubectl get pods -n wordpress
```

Check the logs if needed:

```bash
kubectl logs -l app=wordpress -n wordpress
```
