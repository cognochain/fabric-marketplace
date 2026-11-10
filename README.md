# Wexa Fabric for GCP Marketplace

Helm chart and deployer for the Wexa Fabric GCP Marketplace listing (Kubernetes app, BYOL).

## Prerequisites

- A GKE (or other Kubernetes) cluster, with `kubectl` and `helm` (v3.19+) installed and pointed at it.
- The Kubernetes `Application` CRD, installed once, cluster-wide:
  ```bash
  kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/application/master/config/crd/bases/app.k8s.io_applications.yaml
  ```
- A BYOL licence token from your Wexa contact.

## 1. Create the licence Secret

```bash
kubectl create secret generic wexa-fabric-license \
  --from-literal=token='<token from Wexa>' \
  -n <namespace>
```
The Secret's name (`wexa-fabric-license` above, or anything else you pick) is what you pass as `licensingSecret`, in the GCP Console deployer form, or as `--set license.licensingSecret=<name>` on the CLI.

## 2. Grant IAM roles (one-time, per GCP project)

Every pod authenticates as the project's default Compute Engine service account via the GKE metadata server, no Workload Identity or dedicated service account needed. `deployer/apply.sh` attempts these automatically at install time when run through the Console deployer; it's best-effort, so grant them yourself first if you want a clean install or the installer isn't a project Owner/Editor. Get your project number with `gcloud projects describe <PROJECT_ID> --format='value(projectNumber)'`.

```bash
SA="<PROJECT_NUMBER>-compute@developer.gserviceaccount.com"

gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member="serviceAccount:${SA}" --role="roles/secretmanager.admin"

gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member="serviceAccount:${SA}" --role="roles/aiplatform.user"

gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member="serviceAccount:${SA}" --role="roles/storage.admin"

gcloud iam service-accounts add-iam-policy-binding "${SA}" \
  --member="serviceAccount:${SA}" \
  --role="roles/iam.serviceAccountTokenCreator" \
  --project=<PROJECT_ID>
```

When granting via the Console (IAM & Admin → Service Accounts → the SA → Permissions), watch the role search: "Vertex AI User" has several similarly-named roles that look plausible but are wrong ("Agent Platform User", "Vertex AI Viewer"), and Cloud Storage needs "Storage Admin", not "Storage Object Admin" (the narrower role can't create the bucket this chart needs). The last grant is on a different Console tab: "Principals with access to this service account" → "Grant Access", not "Manage access".

## 3. Install

### From the GCP Console

Configure on the product's Marketplace page: application instance name, namespace, the licence Secret name from step 1, and the ingress hostname you'll point DNS at. Click Deploy.

### From the command line

```bash
helm upgrade --install fabric ./charts/fabric \
  --namespace <namespace> --create-namespace \
  --set license.licensingSecret=wexa-fabric-license \
  --set ingress.host=<your-hostname> \
  --set-string services.model-gateway.vertex.project=<PROJECT_ID> \
  --set-string services.api.env.GCP_PROJECT_ID=<PROJECT_ID> \
  --set-string services.worker.env.GCP_PROJECT_ID=<PROJECT_ID> \
  --set-string services.worker-scheduler.env.GCP_PROJECT_ID=<PROJECT_ID> \
  --set-string services.doc-worker.env.GCP_PROJECT_ID=<PROJECT_ID> \
  --set-string services.api.env.GCS_BUCKET_NAME=<bucket-name> \
  --set-string services.worker.env.GCS_BUCKET_NAME=<bucket-name> \
  --set-string services.worker-scheduler.env.GCS_BUCKET_NAME=<bucket-name> \
  --set-string services.doc-worker.env.GCS_BUCKET_NAME=<bucket-name>
```
No `-f` flag needed: `values.yaml` is already the complete configuration. The `GCP_PROJECT_ID`/`GCS_BUCKET_NAME` overrides above are filled in automatically when installing through the Console deployer; set them yourself only for a direct `helm install`.

## 4. Post-install steps

Two one-time steps are required after every fresh install:

**Unseal the secrets store** (starts sealed; connector credentials can't be read or written until this runs):
```bash
kubectl exec -n <namespace> -it <release>-secrets-0 -- secrets-cli operator init
kubectl exec -n <namespace> -it <release>-secrets-0 -- secrets-cli operator unseal
```
Then set the resulting token as `services.api.env.SECRETS_STORE_TOKEN` and `helm upgrade` again.

**Register the analytics namespace in the workflow engine** (otherwise analytics workers start but silently do nothing):
```bash
kubectl exec -n <namespace> -it deploy/<release>-workflow-engine -- \
  workflow-cli operator namespace create --address localhost:7233 analytics
```

## 5. Verify

```bash
kubectl get pods -n <namespace>
kubectl logs job/<release>-gcp-license-gate -n <namespace>   # should say LICENSE OK
```

## Uninstall

```bash
helm uninstall fabric -n <namespace>
```
StatefulSet PVCs aren't deleted by `helm uninstall` (Kubernetes default); remove them explicitly for a full teardown:
```bash
kubectl delete pvc -n <namespace> -l app.kubernetes.io/instance=fabric
```

## Support

Contact your Wexa representative, or support@wexa.ai.
