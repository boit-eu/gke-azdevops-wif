# Azure DevOps Terraform zu GCP mit WIF

Erstellung von Workload Identity Federation Auth für Azure DevOps mit Google Cloud Platform.

## Variablen

Folgende Variablen wurden definiert

* Workload Identity Pool: azure-pool
* Workload Identity Pool Provider: azure-provider
* Issuer: https://vstoken.dev.azure.com/00000000-0000-0000-0000-000000000000
* Azure DevOps Org ID: 00000000-0000-0000-0000-000000000000
* GCP Service Account Name: terraform-sa
* GCP Projekt: projekt-1234
* GCP SA E-Mail: terraform-sa@projekt-1234.iam.gserviceaccount.com
* Azure DevOps Org Name: ado-org-name
* Azure DevOps Projekt Name: ado-projekt-name
* Azure DevOps Service connection Name: gcp-federation

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
    --member="principalSet://iam.googleapis.com/projects/123456789/locations/global/workloadIdentityPools/azure-pool/attribute.subject/sc://ado-org-name/ado-projekt-name/gcp-federation" \
    --role="roles/iam.workloadIdentityUser"
```

## Setup Azure DevOps

Unter "Project Settings" -> "Service connections" -> "New service connection". Hier den Connection Type "Google Cloud (WIF)" auswählen. Falls dieser nicht zur Auswahl steht, kann er über den Marketplace installiert werden. Die Erweiterung heißt "Google Cloud Auth".

https://marketplace.visualstudio.com/items?itemName=laurensknoll.google-cloud-auth-tasks

In der Service connection sind folgende Angaben zu machen:

* Workload Identity Pool Provider: projects/1234567/locations/global/workloadIdentityPools/azure-pool/providers/azure-provider
* Service account: terraform-sa@projekt-1234.iam.gserviceaccount.com
* Service Connection Name: gcp-federation

Beim Idenity Pool Provider wird die GCP Projektnummer gebraucht. Das ist nicht der Projekt Name. Die Nummer kannst du folgendermaßen herausfinden:

```bash
gcloud projects list
```

Hier wird eine Tabelle mit den Projekten ausgegeben. Aus der Spalte "PROJECT_NUMBER" nimmst du die Projektnummer.

Ersetze ebenfalls die Daten für den Serviceaccount mit deinen erstellen SA. Der Service Connection Name ist ebenfalls wichtig, da dieser bei dem Policy Binding am Service Account angegeben wurde.

### Azure DevOps Organization ID

Um die korrekte Organization ID herauszufinden, gibt es mehrere Wege. Am einfachsten ist es, unter "Service connections" in den "Project settings" einmal eine "Azure Resource Manager" Connection anzulegen. Der "Identity type" kann "App registration (automatic)" bleiben, Credential ist "Workload identity federation". Noch eine Subscription und einen Namespace auswählen. Das ist nicht relevant, wir brauchen nur die Informationen, die anschließend erzeugt werden.

Wenn die Verbindung angelegt wurde, kann unter dem Overview die "Workload Identity federation details" eingesehen werden. Den "Issuer" benötigen wir für die Erstellung des WIF Providers in GCP.

Hier Beispieldaten:

* Issuer: https://vstoken.dev.azure.com/b4c19df1-98c3-4247-90ff-d69023a45233
