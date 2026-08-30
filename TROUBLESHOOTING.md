# Axion Deployment Troubleshooting Guide

Use this guide when a deployment step fails. Run commands from PowerShell unless noted otherwise.

## 0. Identify the Environment First

Always confirm that kubectl is connected to the intended cluster:

```powershell
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
```

For the current environment, the expected values are:

```text
Context: AKS_Practicle
Resource group: B18G24
Region: eastus
ACR: aksacr23.azurecr.io
```

If the context is wrong, connect to AKS:

```powershell
az login
az aks get-credentials `
  --resource-group B18G24 `
  --name AKS_Practicle `
  --overwrite-existing
```

Do not run deployment commands until `kubectl config current-context` shows the intended cluster.

## 1. Quick Health Snapshot

Run this first:

```powershell
kubectl get pods,svc,endpoints,pvc,ingress -o wide
```

Healthy application Pods should show:

```text
1/1 Running
```

Every application Service should have an endpoint. A Service with an empty `ENDPOINTS` column cannot forward traffic.

For detailed status:

```powershell
kubectl get events --sort-by=.lastTimestamp
kubectl get deployments
kubectl get replicasets
```

## 2. Azure Login and Subscription Problems

### Symptom: Azure commands use the wrong subscription

Check the current account:

```powershell
az account show
```

Select the subscription:

```powershell
az account set --subscription "Azure subscription 1"
```

Confirm the cluster exists in the selected subscription:

```powershell
az aks list -o table
```

### Symptom: `az aks get-credentials` fails

Check the exact resource group and cluster name:

```powershell
az aks list `
  --query "[].{Name:name,ResourceGroup:resourceGroup,State:provisioningState}" `
  -o table
```

Then run:

```powershell
az aks get-credentials `
  --resource-group B18G24 `
  --name AKS_Practicle `
  --overwrite-existing
```

## 3. ACR Image Problems

### Symptom: `ErrImagePull` or `ImagePullBackOff`

Inspect the exact error:

```powershell
kubectl describe pod <POD_NAME>
```

Look under `Events` for the image Kubernetes tried to pull.

A local image such as:

```text
dbingestion:v1
```

is not available to AKS. AKS will try Docker Hub and fail if that repository is private or nonexistent.

Use the complete ACR name:

```text
aksacr23.azurecr.io/dbingestion:v1
```

### Confirm local Docker authentication

```powershell
az acr login --name aksacr23
az acr show --name aksacr23 --query loginServer -o tsv
```

Expected:

```text
aksacr23.azurecr.io
```

### Confirm AKS has pull permission

```powershell
az aks update `
  --name AKS_Practicle `
  --resource-group B18G24 `
  --attach-acr aksacr23
```

### Confirm the image exists in ACR

```powershell
az acr repository list --name aksacr23 -o table
az acr repository show-tags --name aksacr23 --repository dbingestion -o table
```

### Build and push correctly

```powershell
cd D:\Work\K8s\axion_system\axion-ingestion-service
docker build -t aksacr23.azurecr.io/dbingestion:v1 .
docker push aksacr23.azurecr.io/dbingestion:v1
```

Use the corresponding repository for telemetry, simulator, and UI:

```text
aksacr23.azurecr.io/telemetryquery:v1
aksacr23.azurecr.io/axion-data-simulator:v1
aksacr23.azurecr.io/axionui:v1
```

After changing an image:

```powershell
kubectl apply -f <DEPLOYMENT_FILE>
kubectl rollout status deployment/<DEPLOYMENT_NAME>
```

### Symptom: UI Docker build says `node.exe not found`

This means Windows `node_modules` was copied into the Linux Docker build. Confirm [axion-ui/.dockerignore](axion-ui/.dockerignore) contains:

```text
node_modules
dist
.git
.github
*.log
```

Then rebuild:

```powershell
cd D:\Work\K8s\axion_system\axion-ui
npm install
npm run build
docker build -t aksacr23.azurecr.io/axionui:v1 .
```

## 4. PostgreSQL Startup Problems

### Symptom: `Database is uninitialized and superuser password is not specified`

The PostgreSQL Deployment is missing a non-empty `POSTGRES_PASSWORD`.

Check the live Deployment:

```powershell
kubectl get deployment postgress-deployment -o yaml
```

The development configuration requires:

```yaml
env:
  - name: POSTGRES_DB
    value: mydb
  - name: POSTGRES_USER
    value: myuser
  - name: POSTGRES_PASSWORD
    value: root@12345
```

Apply the Deployment again:

```powershell
kubectl apply -f D:\Work\K8s\axion_system\axion-database-schema\deployment_postgress.yaml
```

### Symptom: PostgreSQL Pod is `CrashLoopBackOff` or `Error`

Read the previous container log:

```powershell
kubectl logs pod/<POSTGRES_POD_NAME> --previous
```

Describe the Pod:

```powershell
kubectl describe pod <POSTGRES_POD_NAME>
```

### Symptom: `initdb: directory ... exists but is not empty` and `lost+found`

An Azure managed disk contains a `lost+found` directory. PostgreSQL must use a subdirectory.

The Deployment needs:

```yaml
env:
  - name: PGDATA
    value: /var/lib/postgresql/data/pgdata
```

The PVC should be mounted at:

```text
/var/lib/postgresql/data
```

Apply the corrected Deployment:

```powershell
kubectl apply -f D:\Work\K8s\axion_system\axion-database-schema\deployment_postgress.yaml
kubectl rollout status deployment/postgress-deployment --timeout=180s
```

### Symptom: PostgreSQL Service has no endpoint

```powershell
kubectl get pods -l app=postgres
kubectl get endpoints postgress-service
```

The PostgreSQL Pod must be ready, and the Service selector must match:

```yaml
selector:
  app: postgres
```

The Pod label must also be:

```yaml
labels:
  app: postgres
```

## 5. Persistent Volume Problems

### Symptom: PVC is `Pending`

```powershell
kubectl get pvc
kubectl describe pvc postgres-pvc
kubectl get storageclass
```

The current cluster has these usable classes:

```text
default
managed-csi
```

A `WaitForFirstConsumer` message can be normal when the claim has not yet been used by a scheduled Pod.

### Symptom: PVC spec update is forbidden

PVC fields such as StorageClass and access mode cannot normally be changed after creation. You may see:

```text
spec is immutable after creation
```

Do not repeatedly apply a manifest that changes an existing claim from `1Gi/default` to another StorageClass. Either keep the existing claim or create a new claim with a new name after taking a backup.

### Verify that PostgreSQL actually uses the PVC

```powershell
kubectl get pvc postgres-pvc
kubectl describe pod -l app=postgres
kubectl describe deployment postgress-deployment
```

The Deployment must show:

```text
/var/lib/postgresql/data from postgres-data
```

and:

```text
ClaimName: postgres-pvc
```

### Verify data survives a Pod restart

First record the count:

```powershell
$pod = kubectl get pods -l app=postgres -o jsonpath='{.items[0].metadata.name}'
kubectl exec $pod -- psql -U myuser -d mydb -c "SELECT COUNT(*) FROM telemetry;"
```

Restart PostgreSQL:

```powershell
kubectl delete pod -l app=postgres
kubectl wait --for=condition=ready pod -l app=postgres --timeout=180s
```

Check again:

```powershell
$pod = kubectl get pods -l app=postgres -o jsonpath='{.items[0].metadata.name}'
kubectl exec $pod -- psql -U myuser -d mydb -c "SELECT COUNT(*) FROM telemetry;"
```

The count should remain and then increase as the simulator writes new rows.

## 6. Database Schema and Data Problems

### Symptom: API returns `relation "telemetry" does not exist`

The API is reaching PostgreSQL, but the selected database does not contain the table.

Check the database:

```powershell
$pod = kubectl get pods -l app=postgres -o jsonpath='{.items[0].metadata.name}'
kubectl exec $pod -- psql -U myuser -d mydb -c '\dt'
```

Apply the schema in order:

```powershell
cd D:\Work\K8s\axion_system\axion-database-schema
Get-Content -Raw .\01-extensions.sql | kubectl exec -i $pod -- psql -U myuser -d mydb
Get-Content -Raw .\02-telemetry.sql | kubectl exec -i $pod -- psql -U myuser -d mydb
```

Verify:

```powershell
kubectl exec $pod -- psql -U myuser -d mydb -c '\dt'
kubectl exec $pod -- psql -U myuser -d mydb -c "SELECT COUNT(*) FROM telemetry;"
```

### Symptom: API returns zero rows

This is not necessarily an error. Check whether the simulator is running:

```powershell
kubectl get deployment axion-data-simulator
kubectl logs deployment/axion-data-simulator --tail=30
```

Check the latest timestamp:

```powershell
kubectl exec $pod -- psql -U myuser -d mydb -c `
  "SELECT COUNT(*) AS rows, MAX(timestamp) AS latest FROM telemetry;"
```

### Symptom: Previous telemetry rows disappeared

This normally means PostgreSQL was running on ephemeral storage or a new empty PVC. Restore a backup:

```powershell
Get-Content -Raw .\mydb-backup.sql | kubectl exec -i $pod -- psql -U myuser -d mydb
```

Create backups before database changes:

```powershell
kubectl exec $pod -- pg_dump -U myuser -d mydb > .\mydb-backup.sql
```

## 7. PostgreSQL Connection Problems

### Current development connection

```text
postgresql://myuser:root%4012345@postgress-service:8081/mydb
```

The password character `@` is encoded as `%40` in the URL.

### Symptom: `connection refused`

Check the database Pod and Service:

```powershell
kubectl get pod -l app=postgres
kubectl get endpoints postgress-service
```

The endpoint should point to PostgreSQL port `5432`.

Check application environment variables:

```powershell
kubectl get deployment telemetry-query-deployment -o yaml
kubectl get deployment ingestion-deployment -o yaml
```

Both applications must use the actual Service name and Service port. Do not use `localhost` inside a Kubernetes Pod.

### Symptom: `password authentication failed`

Confirm the PostgreSQL values in the Deployment and the application connection string match:

```text
Database: mydb
User: myuser
Password: root@12345
Service: postgress-service
Port: 8081
```

## 8. Ingestion and Simulator Problems

### Symptom: Simulator Deployment is not found

Apply the simulator manifest:

```powershell
cd D:\Work\K8s\axion_system\axion-data-simulator
kubectl apply -f deployment.yaml
kubectl rollout status deployment/axion-data-simulator
```

### Symptom: Simulator logs show failed POST requests

```powershell
kubectl logs deployment/axion-data-simulator --tail=50
kubectl get endpoints ingestion-service
```

The simulator URL should be:

```text
http://ingestion-service:8000/api/v1/telemetry/ingest
```

The ingestion Service must select the Pod:

```yaml
selector:
  app: ingestion
```

The ingestion Deployment must use the ACR image:

```text
aksacr23.azurecr.io/dbingestion:v1
```

### Symptom: Ingestion returns HTTP 500

Read the ingestion logs:

```powershell
kubectl logs deployment/ingestion-deployment --tail=50
```

Common causes:

- PostgreSQL is not ready.
- The `DATABASE_URL` points to the wrong Service, port, user, or database.
- The `telemetry` table has not been created.
- The payload does not match the API model.

A successful request appears as:

```text
201 Created
```

## 9. Telemetry API Problems

### Test internally first

This bypasses DNS and Application Gateway:

```powershell
kubectl port-forward service/telemetry-service 8000:8080
```

In another terminal:

```powershell
curl.exe -i http://localhost:8000/docs
curl.exe -i http://localhost:8000/dashboard/summary
```

Expected:

```text
HTTP/1.1 200 OK
```

### Symptom: Internal API works but public API returns 502

Check:

```powershell
kubectl get endpoints telemetry-service
kubectl describe ingress telemetry-api-ingress
kubectl logs deployment/telemetry-query-deployment --tail=50
```

The Application Gateway health probe uses:

```text
/docs
```

That path must return HTTP 200. Check the backend directly:

```powershell
kubectl port-forward service/telemetry-service 8000:8080
curl.exe -i http://localhost:8000/docs
```

### Symptom: API returns 404

Confirm the requested route. Current routes include:

```text
/dashboard/summary
/devices
/dashboard/throughput
/dashboard/regions
/devices/{device_id}/latest
/devices/{device_id}/trends
/devices/top-anomalous
```

`/` is not an application route and may return 404. `/docs` is the API documentation route.

## 10. Application Gateway Ingress Problems

### Symptom: Ingress is not found

```powershell
kubectl get ingress -A
kubectl get ingressclass
```

Apply the manifest:

```powershell
kubectl apply -f D:\Work\K8s\axion_system\axion-telemetry-query-service\application-gateway-ingress.yaml
```

### Symptom: Ingress has no address

```powershell
kubectl describe ingress telemetry-api-ingress
kubectl get events -A --sort-by=.lastTimestamp
```

Confirm the class exists:

```text
azure-application-gateway
```

The manifest must contain:

```yaml
ingressClassName: azure-application-gateway
```

Confirm the AKS Application Gateway addon is enabled:

```powershell
az aks show `
  --name AKS_Practicle `
  --resource-group B18G24 `
  --query addonProfiles.ingressApplicationGateway -o json
```

### Symptom: Ingress routes to the wrong application

Verify Service names and ports:

```powershell
kubectl describe ingress telemetry-api-ingress
kubectl get svc axion-ui telemetry-service
kubectl get endpoints axion-ui telemetry-service
```

Current routes:

```text
api.dny-ai.store -> telemetry-service:8080 -> Pod:8000
ui.dny-ai.store  -> axion-ui:8080 -> Pod:80
```

## 11. DNS Problems

### Symptom: `api.dny-ai.store` returns NXDOMAIN

At Hostinger DNS for `dny-ai.store`, create:

```text
Type: A
Name: api
Value: 20.102.18.195
```

Verify:

```powershell
nslookup api.dny-ai.store 8.8.8.8
```

### Symptom: `ui.dny-ai.store` returns NXDOMAIN

At Hostinger DNS for `dny-ai.store`, create:

```text
Type: A
Name: ui
Value: 20.102.18.195
```

Enter only `ui` in the Hostinger Name field, not `ui.dny-ai.store` if Hostinger automatically appends the domain.

Verify:

```powershell
nslookup ui.dny-ai.store 8.8.8.8
```

### Symptom: Azure DNS record exists but public DNS shows another provider

Check the authoritative nameservers:

```powershell
nslookup -type=ns aks.com 8.8.8.8
```

If it returns WorldNIC nameservers, Azure DNS is not authoritative. At the registrar for `aks.com`, set:

```text
ns1-04.azure-dns.com
ns2-04.azure-dns.net
ns3-04.azure-dns.org
ns4-04.azure-dns.info
```

Then verify again. The Azure record itself can be checked with:

```powershell
az network dns record-set a show `
  --resource-group b18g24 `
  --zone-name aks.com `
  --name api `
  -o json
```

### Symptom: DNS resolves but browser still fails

Test the gateway directly:

```powershell
curl.exe -i http://20.102.18.195/ -H "Host: ui.dny-ai.store"
curl.exe -i http://20.102.18.195/dashboard/summary -H "Host: api.dny-ai.store"
```

If direct gateway access works but the hostname does not, the issue is DNS or DNS cache:

```powershell
ipconfig /flushdns
```

## 12. UI Problems

### Symptom: UI Pod is not found

```powershell
kubectl get deployment axion-ui
kubectl apply -f D:\Work\K8s\axion_system\axion-ui\deployment.yaml
kubectl apply -f D:\Work\K8s\axion_system\axion-ui\service.yaml
```

### Symptom: UI is reachable but shows no data

The browser calls the API hostname compiled into the UI image. Check the source files:

```text
axion-ui/src/App.tsx
axion-ui/src/components/pages/DashboardView.tsx
axion-ui/src/components/pages/HistoricalTrends.tsx
```

The current API URL should be:

```text
http://api.dny-ai.store
```

If the UI was built before the hostname change, rebuild and push the image:

```powershell
cd D:\Work\K8s\axion_system\axion-ui
npm run build
docker build -t aksacr23.azurecr.io/axionui:v1 .
docker push aksacr23.azurecr.io/axionui:v1
kubectl rollout restart deployment/axion-ui
kubectl rollout status deployment/axion-ui
```

### Symptom: UI loads but API requests fail in the browser

Check the browser developer console for:

- CORS errors
- Mixed-content errors
- Wrong API hostname
- Failed DNS resolution
- HTTP 502 or 500 responses

The current deployment uses HTTP. If you open the UI over HTTPS while the API URL is HTTP, browsers may block the API request as mixed content. Configure HTTPS on Application Gateway before changing the frontend URL to HTTPS.

### Symptom: UI returns `default backend - 404`

Confirm the browser hostname exactly matches an Ingress host:

```text
ui.dny-ai.store
```

Check:

```powershell
kubectl describe ingress telemetry-api-ingress
```

Do not use a hostname that is absent from the Ingress rules.

## 13. pgAdmin Problems

### Access pgAdmin locally

The pgAdmin Service is internal:

```powershell
kubectl port-forward service/pgadmin-service 8080:8080
```

Open:

```text
http://localhost:8080
```

Current development login:

```text
Email: admin@admin.com
Password: Root@12345
```

### Connect pgAdmin to PostgreSQL

Use these values in the pgAdmin server connection dialog:

```text
Host: postgress-service
Port: 8081
Database: mydb
Username: myuser
Password: root@12345
```

Do not use `localhost` in pgAdmin for a PostgreSQL Pod. In pgAdmin, `localhost` means the pgAdmin container itself.

### Symptom: pgAdmin connection times out

Check the database Service and endpoint:

```powershell
kubectl get svc postgress-service
kubectl get endpoints postgress-service
```

Expected endpoint:

```text
<POSTGRES_POD_IP>:5432
```

Use the Service port shown by `kubectl get svc`. In the current development setup, that port is `8081`.

### Symptom: pgAdmin Pod is not ready

```powershell
kubectl logs deployment/pgadmin-deployment --tail=100
kubectl describe pod -l app=pgadmin
```

Check that the required variables exist:

```yaml
PGADMIN_DEFAULT_EMAIL
PGADMIN_DEFAULT_PASSWORD
```

## 14. Egress and Network Policy Problems

### Current dev behavior

The AKS cluster uses outbound type `loadBalancer`, so a separate egress controller is not required for this application.

The simulator should use internal DNS:

```text
http://ingestion-service:8000/api/v1/telemetry/ingest
```

### Symptom: External outbound request fails

Check:

```powershell
kubectl exec deployment/axion-data-simulator -- printenv API_URL
kubectl get networkpolicy -A
az aks show --name AKS_Practicle --resource-group B18G24 --query networkProfile -o json
```

A DNS name is not an egress rule. For domain-based outbound filtering, use Azure Firewall application/FQDN rules. Kubernetes NetworkPolicy generally filters IP/port traffic, not reliable domain names.

## 15. Final Verification Script

Run this after any repair:

```powershell
kubectl get pods,svc,endpoints,pvc,ingress -o wide
nslookup api.dny-ai.store 8.8.8.8
nslookup ui.dny-ai.store 8.8.8.8
curl.exe -i http://api.dny-ai.store/docs
curl.exe -i http://api.dny-ai.store/dashboard/summary
curl.exe -i http://20.102.18.195/ -H "Host: ui.dny-ai.store"
```

Check telemetry growth:

```powershell
$pod = kubectl get pods -l app=postgres -o jsonpath='{.items[0].metadata.name}'
kubectl exec $pod -- psql -U myuser -d mydb -c `
  "SELECT COUNT(*) AS rows, COUNT(DISTINCT device_id) AS devices, MAX(timestamp) AS latest FROM telemetry;"
```

Run the database query again after 10 seconds. The row count and latest timestamp should advance.

## 16. Deployment Recovery Order

When several things are broken, repair them in this order:

1. Confirm Azure subscription and kubectl context.
2. Confirm ACR login and AKS ACR pull permission.
3. Confirm PostgreSQL Pod is ready.
4. Confirm PostgreSQL PVC is `Bound` and mounted.
5. Confirm the `telemetry` table exists.
6. Confirm PostgreSQL Service has an endpoint.
7. Confirm ingestion and telemetry Deployments connect to `postgress-service:8081`.
8. Confirm ingestion Service has an endpoint.
9. Confirm simulator is running and returns HTTP `201 Created`.
10. Confirm telemetry API works with port-forwarding.
11. Confirm telemetry Service has an endpoint.
12. Apply and inspect Application Gateway Ingress.
13. Confirm DNS records resolve to `20.102.18.195`.
14. Confirm the public API returns HTTP `200`.
15. Build, push, and deploy the UI image.
16. Confirm UI Service endpoint and UI Ingress route.
17. Add `ui.dny-ai.store` DNS and open the UI.
18. Configure HTTPS only after HTTP is working.

## 17. Useful Cleanup Commands

Use these carefully. They affect running workloads or data.

Restart an application without deleting its Service:

```powershell
kubectl rollout restart deployment/<DEPLOYMENT_NAME>
kubectl rollout status deployment/<DEPLOYMENT_NAME>
```

Remove a failed Pod and let its Deployment recreate it:

```powershell
kubectl delete pod <POD_NAME>
```

Do not delete `postgres-pvc` unless you have a verified backup and intentionally want to remove persistent database storage.

Never use these commands casually:

```powershell
kubectl delete pvc postgres-pvc
kubectl delete pv <PV_NAME>
kubectl delete deployment postgress-deployment
```

## 18. Known Current Limitations

- Development credentials are present directly in YAML and source defaults.
- PostgreSQL uses one replica.
- The PVC is 1Gi.
- HTTP is used; HTTPS is not configured.
- The `postgress-service` spelling and port `8081` are retained for compatibility.
- The old `AxionUI_Varun_class` tree contains conflicting manifests and should not be applied.
- The current simulator is a test-data generator.
- The API has no authentication beyond the application behavior.
