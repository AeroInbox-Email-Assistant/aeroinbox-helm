# AeroInbox Kubernetes Configuration (Helm Charts & GitOps)

This repository contains the Helm charts, environment values, and ArgoCD application manifests for deploying the AeroInbox platform to Azure Kubernetes Service (AKS).

---

## Folder Structure

```
aeroinbox-helm/
+-- argocd/
¦   +-- applications/
¦       +-- aeroinbox.yaml      # ArgoCD Application definition
+-- charts/
¦   +-- shared/                 # Shared components template (Ingress, SecretProviderClass, ServiceAccount)
¦   +-- api-service/            # API Gateway service templates & HPA
¦   +-- gmail-service/          # Gmail service templates & HPA
¦   +-- ai-service/             # AI service templates
¦   +-- rule-engine/            # Rule Engine templates
¦   +-- meeting-service/        # Meeting service templates & KEDA ScaledObject
¦   +-- frontend/               # Frontend Nginx service templates
+-- environments/
¦   +-- production/
¦   ¦   +-- values.yaml         # Production-specific values & Git HEAD image tags
¦   +-- dev/
¦       +-- values.yaml         # Development environment overrides
+-- values.yaml                 # Global default Helm values
```

---

## Deployment Workflow (GitOps via ArgoCD)

The deployment pipeline is fully automated using GitOps:
1. Code changes are pushed to the `aeroinbox-app` repository.
2. GitHub Actions builds the Docker images and pushes them to Azure Container Registry (ACR), tagging them with the HEAD git commit SHA.
3. The build workflow updates the corresponding image tag in `environments/production/values.yaml` in this repository and pushes the change.
4. **ArgoCD** (running in the AKS namespace `argocd`) polls this repository every 3 minutes.
5. Once a tag mismatch is detected, ArgoCD automatically applies a rolling update to the cluster workloads.

### Immediate Sync Command
If you don't want to wait for the 3-minute poll interval, you can run this command from the **Bastion Jumpbox VM** to force ArgoCD to sync immediately:

```bash
kubectl annotate application aeroinbox -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

---

## Autoscaling Details

- **HTTP Services (`api-service`, `gmail-service`)**: Utilize native Kubernetes **Horizontal Pod Autoscalers (HPA)** based on CPU utilization exceeding 70% (`minReplicas: 2`, `maxReplicas: 5`).
- **Background Worker (`meeting-service`)**: Uses **KEDA (Kubernetes Event-driven Autoscaling)** to scale dynamically based on the queue depth of the Azure Service Bus `meeting-reminders` queue.
  - The KEDA configuration is defined in [`charts/meeting-service/templates/keda-scaler.yaml`](charts/meeting-service/templates/keda-scaler.yaml).
  - Maximum replicas are capped in the `values.yaml` under `meeting-service.autoscaling.maxReplicas`.

---

## Private Cluster & Secrets Management

- **Access**: The production AKS API server has private endpoints. All `kubectl` and administrative commands must be executed from the **Bastion Jumpbox VM**.
- **Secrets Provider**: Secrets (database connection details, OpenAI/Gemini API keys, and Service Bus strings) are mounted directly from Azure Key Vault using **Azure Workload Identity** and the Secrets Store CSI driver (`SecretProviderClass`).
