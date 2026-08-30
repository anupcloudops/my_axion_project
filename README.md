# Axion Platform: AKS Deployment Guide

This guide describes how to recreate the Axion telemetry application on Azure Kubernetes Service (AKS).

The application contains:

- `axion-ui`: React dashboard served by Nginx
- `axion-telemetry-query-service`: FastAPI read/query API
- `axion-ingestion-service`: FastAPI write/ingestion API
- `axion-data-simulator`: background process that generates telemetry
- `axion-database-schema`: PostgreSQL schema and database manifests

This guide uses the current development deployment configuration. It is intentionally simple and is not a production hardening guide.

For symptom-based diagnosis and recovery commands, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

## 1. Architecture

```text
Browser
  |
  | http://ui.dny-ai.store
  v
Azure Application Gateway Ingress
  |
  +--> axion-ui:8080 --> UI Pod:80
  |
  +--> telemetry-service:8080 --> Telemetry API Pod:8000
                                      |
                                      v
                           postgress-service:8081
                                      |
                                      v
                              PostgreSQL Pod:5432

axion-data-simulator
  |
  +--> http://ingestion-service:8000/api/v1/telemetry/ingest
                                      |
                                      v
                              PostgreSQL Pod:5432
```

The simulator and ingestion API communicate internally inside Kubernetes. PostgreSQL, pgAdmin, and ingestion are not exposed through the public Ingress.

## 2. Current Azure and Kubernetes Values

These are the values used during the working deployment:

```text
Azure subscription: Azure subscription 1
AKS cluster:       AKS_Practicle
AKS resource group: B18G24
AKS region:        eastus
ACR:               aksacr23.azurecr.io
Ingress class:     azure-application-gateway
Application Gateway IP: 20.102.18.195
API hostname:      api.dny-ai.store
UI hostname:       ui.dny-ai.store
```

Do not copy these values blindly into another environment. Replace them where necessary.

## 3. Repository Layout

```text
axion_system/
├── README.md
├── axion-data-simulator/
│   ├── Dockerfile
│   ├── deployment.yaml
│   └── simulator.py
├── axion-database-schema/
│   ├── 01-extensions.sql
│   ├── 02-telemetry.sql
│   ├── deployment_postgress.yaml
│   ├── postgres_pvc.yaml
│   ├── postgress_service.yaml
│   ├── deployement_pgadmin.yaml
│   └── pgadmin_service.yaml
├── axion-ingestion-service/
│   ├── Dockerfile
│   ├── deployment_ingestion.yaml
│   └── ingestion_service.yaml
├── axion-telemetry-query-service/
│   ├── Dockerfile
│   ├── deployment_telemetry.yaml
│   ├── telemetry-service.yaml
│   └── application-gateway-ingress.yaml
└── axion-ui/
    ├── Dockerfile
    ├── .dockerignore
    ├── deployment.yaml
    ├── service.yaml
    └── src/
```

## 4. Prerequisites

Install and authenticate these tools:

- Azure CLI (`az`)
- kubectl
- Docker Desktop or another Docker engine
- Node.js and npm for local UI validation

Confirm the tools:

```powershell
az version
kubectl version --client
 docker version
node --version
npm --version
```

Authenticate to Azure:

```powershell
az login
az account show
```

Select the correct subscription if required:

```powershell
az account set --subscription "Azure subscription 1"
```

Connect kubectl to AKS:

```powershell
az aks get-credentials `
  --resource-group B18G24 `
  --name AKS_Practicle `
  --overwrite-existing
```

Confirm the cluster:

```powershell
kubectl config current-context
kubectl get nodes
```

Expected context:

```text
AKS_Practicle
```

## 5. Configure ACR Access

Log in to the registry:

```powershell
az acr login --name aksacr23
```

Confirm its login server:

```powershell
az acr show `
  --name aksacr23 `
  --query loginServer `
  --output tsv
```

Expected:

```text
aksacr23.azurecr.io
```

Allow AKS to pull private images from ACR:

```powershell
az aks update `
  --name AKS_Practicle `
  --resource-group B18G24 `
  --attach-acr aksacr23
```

The local `az acr login` authenticates Docker. The `--attach-acr` operation grants the AKS kubelet identity permission to pull images.

## 6. Build and Push Images

Always use the image name in the corresponding Kubernetes Deployment.

### Ingestion API

```powershell
cd D:\Work\K8s\axion_system\axion-ingestion-service
docker build -t aksacr23.azurecr.io/dbingestion:v1 .
docker push aksacr23.azurecr.io/dbingestion:v1
```

### Telemetry Query API

```powershell
cd D:\Work\K8s\axion_system\axion-telemetry-query-service
docker build -t aksacr23.azurecr.io/telemetryquery:v1 .
docker push aksacr23.azurecr.io/telemetryquery:v1
```

### Data Simulator

```powershell
cd D:\Work\K8s\axion_system\axion-data-simulator
docker build -t aksacr23.azurecr.io/axion-data-simulator:v1 .
docker push aksacr23.azurecr.io/axion-data-simulator:v1
```

### UI

The UI has a `.dockerignore` file. It is important because Windows `node_modules` must not be copied into the Linux Docker build.

```powershell
cd D:\Work\K8s\axion_system\axion-ui
npm install
npm run build
docker build -t aksacr23.azurecr.io/axionui:v1 .
docker push aksacr23.azurecr.io/axionui:v1
```

The UI build must complete before pushing the image.

## 7. PostgreSQL Deployment

The database values used by the current development manifests are:

```text
Database: mydb
Username: myuser
Password: root@12345
Service: postgress-service
Service port: 8081
Container port: 5432
```

The `@` in the password must be URL-encoded as `%40` inside a PostgreSQL connection URL:

```text
postgresql://myuser:root%4012345@postgress-service:8081/mydb
```

### Persistent storage

The PostgreSQL Deployment mounts `postgres-pvc` at:

```text
/var/lib/postgresql/data
```

It also sets:

```text
PGDATA=/var/lib/postgresql/data/pgdata
```

The subdirectory is necessary because Azure managed disk filesystems can contain `lost+found`, and PostgreSQL cannot initialize directly in a non-empty mount root.

Apply the PVC and PostgreSQL resources:

```powershell
cd D:\Work\K8s\axion_system\axion-database-schema
kubectl apply -f postgres_pvc.yaml
kubectl apply -f deployment_postgress.yaml
kubectl apply -f postgress_service.yaml
```

Check the PVC and Pod:

```powershell
kubectl get pvc
kubectl get pods -l app=postgres -o wide
kubectl get endpoints postgress-service
```

Expected PVC state:

```text
Bound
```

The current development PVC is 1Gi and uses the cluster default StorageClass. Do not change the StorageClass or size of an existing PVC in place; PVC fields such as these are generally immutable after creation. Create a new claim if you need a different storage configuration.

## 8. Initialize the Database Schema

The schema files must be applied in order:

```text
01-extensions.sql
02-telemetry.sql
```

Find the current PostgreSQL Pod:

```powershell
$pod = kubectl get pods -l app=postgres -o jsonpath='{.items[0].metadata.name}'
```

Apply the extension:

```powershell
Get-Content -Raw .\01-extensions.sql | kubectl exec -i $pod -- psql -U myuser -d mydb
```

Apply the telemetry table:

```powershell
Get-Content -Raw .\02-telemetry.sql | kubectl exec -i $pod -- psql -U myuser -d mydb
```

Verify the table:

```powershell
kubectl exec $pod -- psql -U myuser -d mydb -c '\dt'
```

Expected table:

```text
public | telemetry | table | myuser
```

## 9. Backup and Restore

Create a backup before database changes:

```powershell
$pod = kubectl get pods -l app=postgres -o jsonpath='{.items[0].metadata.name}'
kubectl exec $pod -- pg_dump -U myuser -d mydb > .\mydb-backup.sql
```

Restore a backup:

```powershell
$pod = kubectl get pods -l app=postgres -o jsonpath='{.items[0].metadata.name}'
Get-Content -Raw .\mydb-backup.sql | kubectl exec -i $pod -- psql -U myuser -d mydb
```

Verify data:

```powershell
kubectl exec $pod -- psql -U myuser -d mydb -c `
  "SELECT COUNT(*) AS rows, MAX(timestamp) AS latest FROM telemetry;"
```

## 10. pgAdmin Deployment

Apply pgAdmin:

```powershell
cd D:\Work\K8s\axion_system\axion-database-schema
kubectl apply -f deployement_pgadmin.yaml
kubectl apply -f pgadmin_service.yaml
```

Current pgAdmin login:

```text
Email: admin@admin.com
Password: Root@12345
```

The pgAdmin Service is internal. Open it locally with:

```powershell
kubectl port-forward service/pgadmin-service 8080:8080
```

Open in a browser:

```text
http://localhost:8080
```

Create a pgAdmin server connection with:

```text
Host: postgress-service
Port: 8081
Database: mydb
Username: myuser
Password: root@12345
```

Do not expose PostgreSQL publicly. pgAdmin can be exposed through an Ingress later if required, but it should normally remain internal.

## 11. Deploy the Ingestion API

The ingestion Deployment uses:

```text
aksacr23.azurecr.io/dbingestion:v1
```

Apply it:

```powershell
cd D:\Work\K8s\axion_system\axion-ingestion-service
kubectl apply -f deployment_ingestion.yaml
kubectl apply -f ingestion_service.yaml
kubectl rollout status deployment/ingestion-deployment
```

The Service should be internal:

```yaml
type: ClusterIP
```

Verify:

```powershell
kubectl get deployment ingestion-deployment
kubectl get service ingestion-service
kubectl get endpoints ingestion-service
kubectl logs deployment/ingestion-deployment --tail=30
```

The application should connect to PostgreSQL using:

```text
postgresql://myuser:root%4012345@postgress-service:8081/mydb
```

If the Deployment does not define `DATABASE_URL`, it uses the image's default configuration. For predictable deployment, define `DATABASE_URL` in the Deployment or, preferably, a Kubernetes Secret.

## 12. Deploy the Telemetry Query API

The telemetry Deployment uses:

```text
aksacr23.azurecr.io/telemetryquery:v1
```

Apply it:

```powershell
cd D:\Work\K8s\axion_system\axion-telemetry-query-service
kubectl apply -f deployment_telemetry.yaml
kubectl apply -f telemetry-service.yaml
kubectl rollout status deployment/telemetry-query-deployment
```

The Service mapping is:

```text
telemetry-service:8080 -> telemetry Pod:8000
```

Verify:

```powershell
kubectl get deployment telemetry-query-deployment
kubectl get service telemetry-service
kubectl get endpoints telemetry-service
kubectl logs deployment/telemetry-query-deployment --tail=30
```

Test internally:

```powershell
kubectl port-forward service/telemetry-service 8000:8080
```

In another terminal:

```powershell
curl.exe http://localhost:8000/docs
curl.exe http://localhost:8000/dashboard/summary
```

## 13. Deploy the Data Simulator

The simulator has no incoming port, so it does not need a Service or Ingress.

Apply it:

```powershell
cd D:\Work\K8s\axion_system\axion-data-simulator
kubectl apply -f deployment.yaml
kubectl rollout status deployment/axion-data-simulator
```

The simulator is configured to call:

```text
http://ingestion-service:8000/api/v1/telemetry/ingest
```

Check its logs:

```powershell
kubectl logs deployment/axion-data-simulator --tail=30
```

Successful ingestion logs contain:

```text
201 Created
```

Verify that data is increasing:

```powershell
$pod = kubectl get pods -l app=postgres -o jsonpath='{.items[0].metadata.name}'
kubectl exec $pod -- psql -U myuser -d mydb -c `
  "SELECT COUNT(*) AS rows, MAX(timestamp) AS latest FROM telemetry;"
```

Run the command again after 10 seconds. The row count should increase.

## 14. Application Gateway Ingress

The cluster already has this Ingress class:

```text
azure-application-gateway
```

The manifest is:

```text
axion-telemetry-query-service/application-gateway-ingress.yaml
```

It routes:

```text
api.aks.com       -> telemetry-service:8080
api.dny-ai.store  -> telemetry-service:8080
ui.dny-ai.store   -> axion-ui:8080
```

Apply it:

```powershell
kubectl apply -f D:\Work\K8s\axion_system\axion-telemetry-query-service\application-gateway-ingress.yaml
```

Verify:

```powershell
kubectl get ingress telemetry-api-ingress -o wide
kubectl describe ingress telemetry-api-ingress
```

The Ingress address should be the Application Gateway IP.

The API health probe uses:

```text
/docs
```

The telemetry API must continue serving `/docs`, otherwise Application Gateway may mark the backend unhealthy.

## 15. Deploy the UI

The UI Deployment uses:

```text
aksacr23.azurecr.io/axionui:v1
```

Apply it:

```powershell
cd D:\Work\K8s\axion_system\axion-ui
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl rollout status deployment/axion-ui
```

The Service mapping is:

```text
axion-ui:8080 -> UI Pod:80
```

Verify:

```powershell
kubectl get deployment axion-ui
kubectl get service axion-ui
kubectl get endpoints axion-ui
kubectl logs deployment/axion-ui --tail=30
```

The UI source currently calls:

```text
http://api.dny-ai.store
```

This is defined in:

```text
axion-ui/src/App.tsx
axion-ui/src/components/pages/DashboardView.tsx
axion-ui/src/components/pages/HistoricalTrends.tsx
```

The UI must use the same API hostname that appears in the Ingress and DNS records.

## 16. Hostinger DNS for `dny-ai.store`

The domain is managed by Hostinger nameservers:

```text
ns1.dns-parking.com
ns2.dns-parking.com
```

In Hostinger DNS records, create:

```text
Type: A
Name: api
Value: 20.102.18.195
TTL: 300 or default
```

This creates:

```text
api.dny-ai.store -> 20.102.18.195
```

Create the UI record:

```text
Type: A
Name: ui
Value: 20.102.18.195
TTL: 300 or default
```

This creates:

```text
ui.dny-ai.store -> 20.102.18.195
```

Do not enter the complete domain in the Hostinger Name field. Enter only `api` or `ui`.

Verify:

```powershell
nslookup api.dny-ai.store 8.8.8.8
nslookup ui.dny-ai.store 8.8.8.8
```

Both should return:

```text
20.102.18.195
```

## 17. Azure Public DNS Zone `aks.com`

Azure contains a Public DNS Zone named `aks.com` in resource group `b18g24`.

Its Azure nameservers are:

```text
ns1-04.azure-dns.com
ns2-04.azure-dns.net
ns3-04.azure-dns.org
ns4-04.azure-dns.info
```

The Azure record is:

```text
api.aks.com A 20.102.18.195
```

However, Azure DNS is used publicly only after the domain registrar delegates `aks.com` to those Azure nameservers. Check delegation:

```powershell
nslookup -type=ns aks.com 8.8.8.8
```

If it returns WorldNIC nameservers, Azure DNS is not authoritative yet. Change the nameservers at the `aks.com` registrar to the four Azure nameservers above.

Do not confuse a Public DNS Zone with internal DNS. For private-only access, create an Azure Private DNS Zone and link it to the AKS VNet.

## 18. HTTP and HTTPS

The current working setup uses HTTP:

```text
http://ui.dny-ai.store
http://api.dny-ai.store/dashboard/summary
```

Test the API:

```powershell
curl.exe http://api.dny-ai.store/dashboard/summary
```

Test the UI directly through the Application Gateway before DNS is available:

```powershell
curl.exe -i http://20.102.18.195/ -H "Host: ui.dny-ai.store"
```

Expected:

```text
HTTP/1.1 200 OK
Server: nginx
```

HTTPS is not enabled by the current Ingress. To enable HTTPS later, configure a certificate on Application Gateway and change the UI API URL to:

```typescript
const API_BASE = 'https://api.dny-ai.store';
```

Then rebuild and push the UI image.

## 19. Egress

The AKS cluster currently uses:

```text
outboundType: loadBalancer
```

This already provides outbound internet access for the dev deployment. A separate egress controller is not required for the current application.

Use Kubernetes internal DNS for application traffic:

```text
ingestion-service:8000
telemetry-service:8080
postgress-service:8081
```

The simulator should not call a public DNS name to reach ingestion.

If outbound traffic to `dny-ai.store` must be restricted by domain, use Azure Firewall FQDN/application rules. A Kubernetes NetworkPolicy is useful for Pod-level traffic control, but it does not reliably implement domain-name egress filtering.

## 20. Complete Deployment Sequence

Run these commands in order:

```powershell
# Connect to AKS
az aks get-credentials --resource-group B18G24 --name AKS_Practicle --overwrite-existing

# Database
cd D:\Work\K8s\axion_system\axion-database-schema
kubectl apply -f postgres_pvc.yaml
kubectl apply -f deployment_postgress.yaml
kubectl apply -f postgress_service.yaml
kubectl apply -f deployement_pgadmin.yaml
kubectl apply -f pgadmin_service.yaml

# Ingestion API
cd D:\Work\K8s\axion_system\axion-ingestion-service
kubectl apply -f deployment_ingestion.yaml
kubectl apply -f ingestion_service.yaml

# Telemetry Query API
cd D:\Work\K8s\axion_system\axion-telemetry-query-service
kubectl apply -f deployment_telemetry.yaml
kubectl apply -f telemetry-service.yaml
kubectl apply -f application-gateway-ingress.yaml

# Simulator
cd D:\Work\K8s\axion_system\axion-data-simulator
kubectl apply -f deployment.yaml

# UI
cd D:\Work\K8s\axion_system\axion-ui
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Final status
kubectl get pods,svc,endpoints,pvc,ingress -o wide
```

Apply the SQL schema after PostgreSQL is ready if the persistent database is empty.

## 21. Health Checklist

Run:

```powershell
kubectl get pods
kubectl get svc
kubectl get endpoints
kubectl get pvc
kubectl get ingress
```

Confirm:

- [ ] All application Pods show `1/1 Running`.
- [ ] `postgres-pvc` shows `Bound`.
- [ ] PostgreSQL Deployment mounts the PVC.
- [ ] PostgreSQL uses `PGDATA=/var/lib/postgresql/data/pgdata`.
- [ ] `postgress-service` has a `5432` Pod endpoint.
- [ ] `telemetry-service` has an endpoint on Pod port `8000`.
- [ ] `ingestion-service` is `ClusterIP`.
- [ ] `axion-ui` has an endpoint on Pod port `80`.
- [ ] Simulator logs show the internal ingestion URL.
- [ ] Ingestion logs show HTTP `201 Created`.
- [ ] The telemetry row count increases.
- [ ] ACR images use `aksacr23.azurecr.io`.
- [ ] AKS has ACR pull access.
- [ ] Ingress class is `azure-application-gateway`.
- [ ] Ingress address is `20.102.18.195`.
- [ ] `api.dny-ai.store` resolves to `20.102.18.195`.
- [ ] `ui.dny-ai.store` resolves to `20.102.18.195`.
- [ ] `curl.exe http://api.dny-ai.store/dashboard/summary` returns HTTP 200.
- [ ] `http://ui.dny-ai.store` loads the React application.

## 22. Troubleshooting

### ImagePullBackOff or ErrImagePull

Check the image name:

```powershell
kubectl describe pod <pod-name>
```

The image must include the ACR hostname, for example:

```text
aksacr23.azurecr.io/axionui:v1
```

Confirm AKS ACR access:

```powershell
az aks update --name AKS_Practicle --resource-group B18G24 --attach-acr aksacr23
```

### No PostgreSQL endpoint

```powershell
kubectl get endpoints postgress-service
kubectl get pods -l app=postgres
```

The Service selector must match:

```yaml
selector:
  app: postgres
```

### Connection refused from applications

Check that PostgreSQL is running and that the application uses:

```text
postgress-service:8081
```

Do not use `localhost` from an application Pod.

### Relation `telemetry` does not exist

Apply the schema:

```powershell
$pod = kubectl get pods -l app=postgres -o jsonpath='{.items[0].metadata.name}'
Get-Content -Raw D:\Work\K8s\axion_system\axion-database-schema\02-telemetry.sql | kubectl exec -i $pod -- psql -U myuser -d mydb
```

### PostgreSQL fails with `lost+found`

Ensure the Deployment contains:

```yaml
env:
  - name: PGDATA
    value: /var/lib/postgresql/data/pgdata
```

### API returns HTTP 502

Check the Ingress and Service:

```powershell
kubectl describe ingress telemetry-api-ingress
kubectl get endpoints telemetry-service
```

Test the backend internally:

```powershell
kubectl port-forward service/telemetry-service 8000:8080
curl.exe http://localhost:8000/dashboard/summary
```

### UI hostname does not resolve

Add the `ui` A record in Hostinger:

```text
ui -> 20.102.18.195
```

Then query:

```powershell
nslookup ui.dny-ai.store 8.8.8.8
```

### Browser shows the UI but no data

Confirm the frontend API hostname matches the Ingress and DNS:

```text
http://api.dny-ai.store
```

Check the browser developer console for CORS, mixed-content, or API errors. The current HTTP setup must be accessed over HTTP. If the UI is opened over HTTPS while the API is HTTP, the browser will block API calls as mixed content.

## 23. Important Development Limitations

The current setup is suitable for development testing, but it has known limitations:

- PostgreSQL credentials are directly present in development manifests.
- pgAdmin credentials are directly present in development manifests.
- PostgreSQL has one replica.
- The current PVC is 1Gi.
- The default StorageClass has a Delete reclaim policy.
- HTTPS is not configured.
- The UI and API currently use HTTP.
- No resource requests or limits are defined on several existing workloads.
- No NetworkPolicy restricts application traffic.
- The simulator is intended for test data only.
- The old duplicate `AxionUI_Varun_class` tree can cause accidental deployment conflicts.

For a later hardening pass, move credentials to Secrets or Key Vault, pin image versions, add probes/resources, configure TLS, add backups, and separate dev/test/prod namespaces.
