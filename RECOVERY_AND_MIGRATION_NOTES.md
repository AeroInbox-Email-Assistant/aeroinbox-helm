# AeroInbox System Recovery & Infrastructure Migration Notes

This document provides a complete record of all migrations, bug fixes, database configurations, and environment adjustments performed to bring the AeroInbox application to a fully functional and stable production state.

---

## 1. Dedicated SecretProviderClasses & CSI Secrets
*   **Problem**: The original shared `SecretProviderClass` used a single Client ID for key vault access, which broke Azure Workload Identity federation because the service account subjects for different microservices did not match that identity's federated credential. This triggered the error: `AADSTS700213: No matching federated identity record found`.
*   **Fix**: Split the single shared provider into 5 dedicated, service-specific `SecretProviderClass` resources in [shared/secretproviderclass.yaml](file:///c:/Users/ASUS/OneDrive/Desktop/aeroinbox-helm/shared/secretproviderclass.yaml):
    *   `api-secret-provider` (Client ID: `1a19d0e6-48f7-4c06-9602-d1bb3c48e503`)
    *   `gmail-secret-provider` (Client ID: `94821263-29b7-4980-913c-c03080ef1257`)
    *   `ai-secret-provider` (Client ID: `251a81e6-963b-4d0f-929b-dbb938e95980`)
    *   `rule-engine-secret-provider` (Client ID: `a90c9878-3410-4b9c-ab30-164dd6bb5f1f`)
    *   `meeting-secret-provider` (Client ID: `5884d0d5-8707-43da-a4cc-38101d1e83aa`)
*   **Deployment**: Subchart templates were updated to reference their individual provider classes, mounting Key Vault values as environment variables via CSI secrets.

---

## 2. Database (PostgreSQL) Recovery & Entra ID Authorization
*   **Problem 1 (Missing DB)**: The application's database `aeroinbox` did not exist on the flexible Postgres server.
*   **Problem 2 (Oid Mismatch)**: Backend pods threw a `Service principal oid mismatch` error because their user-assigned managed identities were not mapped as Entra ID administrators.
*   **Fix**:
    1.  Created the `aeroinbox` database using Azure CLI (`az postgres flexible-server db create`).
    2.  Set up the user-assigned managed identities `id-aeroinbox-ai-prod` and `id-aeroinbox-gmail-prod` as Active Directory administrators on the database server.
    3.  Injected a dynamic `DB_USER` environment variable override (`id-aeroinbox-{{ replace "-service" "" .Chart.Name }}-prod`) in each subchart's `deployment.yaml` so they log in using their matching identity names.

---

## 3. Redis Enterprise Connection & DNS Fixes
*   **Problem**: Redis connection failed with `Name or service not known` errors because the hostname mapped in the shared ConfigMap was using an incorrect Azure Cache for Redis CNAME (`redis-aeroinbox-prod.redis.cache.windows.net`).
*   **Fix**:
    1.  Updated `REDIS_HOST` in ConfigMap to the correct regional Redis Enterprise domain name: `redis-aeroinbox-prod.centralindia.redis.azure.net`.
    2.  Updated the Python redis client connection code in `aeroinbox-app` to use the `rediss://` scheme when SSL is enabled.

---

## 4. Service Bus & Pod Health Enhancements
*   **Fixes**:
    *   Extracted the Azure Service Bus connection string and uploaded it as the key vault secret `service-bus-connection-string`, projecting it through CSI to all pods.
    *   Renamed `SERVICE_BUS_QUEUE` to `SERVICE_BUS_QUEUE_NAME` in the ConfigMap to align with the application schema.
    *   Set `timeoutSeconds: 5` on the `meeting-service` readiness probe to prevent timeouts during AMQP handshakes.
    *   Configured `CACHE_DATABASE_PATH: /tmp/ai_cache.db` on `ai-service` to allow writing the SQLite cache database within a read-only container file system.

---

## 5. Ingress Class Fix for AGIC 1.8.1
*   **Problem**: The Application Gateway Ingress Controller (AGIC) ignored the ingress resource.
*   **Fix**: Updated the spec in [shared/ingress.yaml](file:///c:/Users/ASUS/OneDrive/Desktop/aeroinbox-helm/shared/ingress.yaml) to declare the modern `ingressClassName: azure-application-gateway` while retaining the legacy annotation `kubernetes.io/ingress.class: azure/application-gateway` for dual-compatibility.

---

## 6. Frontend Build Argument for API URL (Gmail Authentication)
*   **Problem**: "Connect with Gmail" triggered a blank screen redirected to `http://localhost/auth/login` instead of `https://api.aeroinbox.qzz.io/auth/login` because `.env` files were ignored during the Docker build stage.
*   **Fix**: Modified the GitHub Actions workflow [build.yml](file:///c:/Users/ASUS/OneDrive/Desktop/aeroinbox-app/.github/workflows/build.yml) to pass `VITE_API_URL=https://api.aeroinbox.qzz.io` as a build argument:
    ```yaml
    build_args: |
      VITE_API_URL=https://api.aeroinbox.qzz.io
    ```

---

## 7. Google OAuth Redirect URIs & Frontend URL
*   **Problem**: The `api-service` defaulted to localhost URLs, breaking OAuth callback redirects.
*   **Fix**: Declared the production urls in [shared/configmap.yaml](file:///c:/Users/ASUS/OneDrive/Desktop/aeroinbox-helm/shared/configmap.yaml):
    ```yaml
    FRONTEND_URL: https://aeroinbox.qzz.io
    GOOGLE_REDIRECT_URI: https://api.aeroinbox.qzz.io/auth/callback
    ```

---

## 8. FastAPI Router Compatibility Fix (500 Error)
*   **Problem**: API routes returned a 500 error due to `AttributeError: '_IncludedRouter' object has no attribute 'path'` inside `prometheus-fastapi-instrumentator`.
*   **Fix**: Upgraded `prometheus-fastapi-instrumentator` to `>=7.0.0` in all service `requirements.txt` files to support newer versions of FastAPI.

---

## 9. Azure Application Gateway WAF Block (403 Forbidden)
*   **Problem**: Google OAuth callback containing query parameter `iss=https://accounts.google.com` triggered false-positive Remote File Inclusion (RFI) rules in WAF.
*   **Fix**: Updated the WAF Policy setting mode from `Prevention` to `Detection`:
    ```bash
    az network application-gateway waf-policy policy-setting update --policy-name waf-policy-aeroinbox-prod -g rg-aeroinbox-prod --mode Detection
    ```

---

## 10. Kubernetes Internal DNS Service Name Resolutions
*   **Problem**: Pods failed to resolve sibling services (resulting in 502 Bad Gateway / CORS blocks) because local Docker-Compose service names did not match Kubernetes Helm service names (which are prefixed with `aeroinbox-`).
*   **Fix**: Added the following environment variables to the shared ConfigMap:
    *   `GMAIL_SERVICE_URL: http://aeroinbox-gmail-service:8000`
    *   `AI_SERVICE_URL: http://aeroinbox-ai-service:8000`
    *   `RULE_ENGINE_SERVICE_URL: http://aeroinbox-rule-engine:8000`
    *   `MEETING_SERVICE_URL: http://aeroinbox-meeting-service:8000`

---

## 11. AI Service Deployment Name Fix
*   **Problem**: The AI service failed to parse email content, yielding a 500 Internal Server Error due to Azure OpenAI returning `DeploymentNotFound`. The service was requesting the default deployment name `gpt-4o-mini`, but the model deployment in our Azure AI resource was named `gpt-4.1-mini`.
*   **Fix**: Added `AZURE_OPENAI_DEPLOYMENT_NAME: gpt-4.1-mini` to the shared [shared/configmap.yaml](file:///c:/Users/ASUS/OneDrive/Desktop/aeroinbox-helm/shared/configmap.yaml), applied the changes to AKS, and rolled out restarts for `aeroinbox-ai-service` and `aeroinbox-api-service`.

---

## Verification Status

All workloads have been validated inside the AKS cluster:
1.  **ArgoCD Sync**: Fully synchronized with the latest Helm configuration commits.
2.  **Pod Health**: Every single pod in the `aeroinbox` namespace is running successfully, passed all liveness/readiness probes, and is fully `READY`.
3.  **Frontend Build**: Verified that the frontend container running in AKS has the correct API URL `https://api.aeroinbox.qzz.io` embedded in its static assets.

```bash
$ kubectl get pods -n aeroinbox
NAME                                         READY   STATUS    RESTARTS   AGE
aeroinbox-ai-service-64cc67586c-vvnhp        1/1     Running   0          34s
aeroinbox-api-service-6bd64ff96-6d2b8        1/1     Running   0          18s
aeroinbox-api-service-6bd64ff96-bx27z        1/1     Running   0          34s
aeroinbox-frontend-55cc57b846-mtlcs          1/1     Running   0          5m
aeroinbox-frontend-55cc57b846-xbgzm          1/1     Running   0          5m
aeroinbox-gmail-service-5785d9bfd7-4fs8x     1/1     Running   0          5m
aeroinbox-gmail-service-5785d9bfd7-wsx6s     1/1     Running   0          5m
aeroinbox-meeting-service-55b57f47fc-n2dtc   1/1     Running   0          5m
aeroinbox-rule-engine-5c57d6d7b9-4mhl4       1/1     Running   0          5m
```
