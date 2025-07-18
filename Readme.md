# Azure DevOps Terraform zu GCP mit WIF

How to setup Workload Identity Federation for deploying resources in GCP with Azure DevOps Pipelines and Terraform.

## Setup Google Accounts

GCP WIF Pool erstellen für Azure DevOps

```bash
gcloud iam workload-identity-pools create azure-pool \
  --location="global" \
  --display-name="Azure DevOps Pool" \
  --description="Pool für Azure DevOps Pipelines"
```

GCP WIF Provider erstellen

ACHTUNG: Beim Issuer wird die ID der Azure DevOps Organization benötigt. Ich erkläre weiter unten, wie man diese am besten herausfindet.

```bash
gcloud iam workload-identity-pools providers create-oidc azure-provider \
  --location="global" \
  --workload-identity-pool="azure-pool" \
  --issuer-uri="https://vstoken.dev.azure.com/00000000-0000-0000-0000-000000000000" \
  --allowed-audiences="api://AzureADTokenExchange" \
  --attribute-mapping="google.subject=assertion.sub"
```

GCP Service Account erstellen

```bash
gcloud iam service-accounts create terraform-sa \
  --display-name="Terraform SA"
  --project="projekt-1234"  # Hier den Projektnamen eintragen
```

Policy Binding für den Service Account

```bash
gcloud iam service-accounts add-iam-policy-binding \
    terraform-sa@projekt-1234.iam.gserviceaccount.com \
    --member="principalSet://iam.googleapis.com/projects/123456789/locations/global/workloadIdentityPools/azure-pool/attribute.subject/sc://<ado-org-name>/<ado-projekt-name>/<ado-serviceconnection>" \
    --role="roles/iam.workloadIdentityUser"
```

## Setup Azure DevOps


