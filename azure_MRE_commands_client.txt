### USER VARIABLES
#### Edit these values for your environment

```
TENANT_ID="<your-tenant-id>"                     # Azure Tenant ID (Directory ID)
SUBSCRIPTION_ID="<your-subscription-id>"         # Azure Subscription ID
LOCATION="eastus"                                # Azure region
RESOURCE_GROUP="csai-mre"                        # Resource group name
ACR_NAME="chatsafetyaimreacr"                    # Container Registry (lowercase, no dashes)
STORAGE_ACCOUNT="chatsafetyaimrestorage"         # Storage Account (lowercase, no dashes)
CONTAINERAPPS_ENV="csai-acaenv-mre"              # Container Apps Environment
LOG_WORKSPACE="csai-mre-logs"                    # Log Analytics workspace name
AISERVICES_NAME="csai-aiservices-mre"            # Azure Cognitive Services resource
ENV_UNIQUE="orangesand-3abf4345"                 # Container Apps env unique string (portal-generated)
COMPANY_NAME="CompanyTest"                       # Company name for dashboard
```

### Log into the Azure Command Line Interface (CLI)

#### Note: tenant ID is also known as directory ID
```
az login --tenant $TENANT_ID
```

#### Note: run "az account list --output table" to list accessible subscriptions
```
az account set --subscription $SUBSCRIPTION_ID
```

## Azure-Managed Services

#### 1) Create resource group
```
az group create --name $RESOURCE_GROUP --location $LOCATION
```

#### 2) Create Azure Container Registry. Note: name cannot contain dashes

##### Register the resource provider for Container Registry
```
az provider register --namespace Microsoft.ContainerRegistry --wait
az provider register --namespace Microsoft.CognitiveServices --wait
```
```
az acr create \
  --name $ACR_NAME \
  --resource-group $RESOURCE_GROUP \
  --sku Standard \
  --admin-enabled true \
  --location $LOCATION
```

#### 3) Create Storage Account. Note: name cannot contain dashes
```
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --sku Standard_ZRS \
  --location $LOCATION \
  --kind StorageV2 \
  --enable-hierarchical-namespace false
```

##### In Azure Portal:
- In "Configuration", select minimum TLS version 1.2 and click "Save"
- In "Containers", add a container named "database", and inside, add a directory named "ChatSafetyAI_user_data"

#### 4) Create AI Foundry multi-service for OpenAI, Content Safety, Computer Vision, and Language
```
az cognitiveservices account create \
  --name $AISERVICES_NAME \
  --resource-group $RESOURCE_GROUP \
  --kind AIServices \
  --sku S0 \
  --location $LOCATION \
  --assign-identity \
  --yes
```

In Azure Portal:
- a) Click "Generate Custom Domain name" to be able to use managed identities. Use the resource name (here, `$AISERVICES_NAME`), as the custom domain name.

- b) Click "go to Azure AI Foundry portal", and on the "Models + endpoints" tab, deploy the following base models:
  - gpt-4.1-2025-04-14
  - gpt-4.1-mini-2025-04-14
  - text-embedding-3-large
  - NOTE: in "Customize":
    - Opt out of automatic model version upgrades !!!
    - Check quotas! If less than 1k requests per minute (RPM) and 1M tokens per minute (TPM), request increase here: https://aka.ms/oai/quotaincrease

- c) In "Guardrails + controls", create a new content filter with all thresholds set to minimum, and jailbreak and indirect attacks set to "annotate only". Do that for input and output, and apply filter to all deployments.

## SafetyAI-Managed Services

#### 5) Upload Docker images to container registry
```
sudo docker login $ACR_NAME.azurecr.io
```
Enter credentials (found in the Azure Portal page of the container registry, under "Access keys").

```
for image in \
  chatsafetyai \
  dashboard \
  safetyai_attribute_detection \
  safetyai_prediction \
  chatsafetyai_utilities
do
  sudo docker tag $image $ACR_NAME.azurecr.io/$image:latest
  sudo docker push $ACR_NAME.azurecr.io/$image:latest
done
```

#### 6) Create Azure Container Apps environment

```
az extension add --name containerapp
az provider register -n Microsoft.OperationalInsights --wait
```

##### a. Create Log Analytics workspace
```
az monitor log-analytics workspace create \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $LOG_WORKSPACE \
  --location $LOCATION
```

##### b. Get workspace ID and key
```
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $LOG_WORKSPACE \
  --query customerId \
  --output tsv)

WORKSPACE_KEY=$(az monitor log-analytics workspace get-shared-keys \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $LOG_WORKSPACE \
  --query primarySharedKey \
  --output tsv)
```

##### c. Create Container Apps environment with system-assigned identity and logging
```
az containerapp env create \
  --name $CONTAINERAPPS_ENV \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --mi-system-assigned \
  --logs-workspace-id $WORKSPACE_ID \
  --logs-workspace-key $WORKSPACE_KEY
```

#### 7) Create services

```
for app in \
  "csai-mre-prediction $ACR_NAME.azurecr.io/safetyai_prediction:latest 4387" \
  "csai-mre-attribute-detection $ACR_NAME.azurecr.io/safetyai_attribute_detection:latest 4388" \
  "csai-mre-utilities $ACR_NAME.azurecr.io/chatsafetyai_utilities:latest 4386" \
  "csai-mre-chatbot $ACR_NAME.azurecr.io/chatsafetyai:latest 3838" \
  "csai-mre-dashboard $ACR_NAME.azurecr.io/dashboard:latest 3838"
do
  set -- $app
  az containerapp create \
    --name $1 \
    --resource-group $RESOURCE_GROUP \
    --environment $CONTAINERAPPS_ENV \
    --image $2 \
    --target-port $3 \
    --ingress external \
    --registry-server $ACR_NAME.azurecr.io \
    --registry-identity system \
    --min-replicas 1 \
    --system-assigned
done
```

#### 8) Set arguments and environment variables needed by each service

NOTE: AZURE_STORAGE_ACCOUNT_KEY should be copied from the "Access keys" tab of the Storage account's page in Azure Portal

##### a. Utilities API

```
az containerapp update \
  --name csai-mre-utilities \
  --resource-group $RESOURCE_GROUP \
  --set-env-vars \
    AZURE_STORAGE_ACCOUNT=$STORAGE_ACCOUNT \
    AZURE_STORAGE_ACCOUNT_CONTAINER_NAME=database \
    USE_MANAGED_IDENTITY=TRUE \
    IS_AZURE=TRUE \
    ADDRESS_AZURE_EMBEDDINGS="https://$AISERVICES_NAME.cognitiveservices.azure.com/openai/deployments/text-embedding-3-large/embeddings?api-version=2023-05-15" \
    AZURE_STORAGE_ACCOUNT_KEY='<your-storage-account-key>'
```

##### b. ChatSafetyAI

##### NOTES:

- Enable session affinity in the app's page on the Azure Portal, under Ingress.

The notes below are just for your knowledge, no action needed in principle with the new commands:

- UTILITIES_ADDRESS, PREDICTIONS_ADDRESS, and NLP_ADDRESS can be found in the respective overview pages of the apps in the Azure Portal ("Application Url") - they must end with a "/"

- AZURE_OPENAI_ADDRESS_GPT4 and AZURE_OPENAI_ADDRESS_GPT35 can be found the "Endpoint" area in the Azure AI Foundry page of the corresponding deployment

- AZURE_MODERATION_ADDRESS, AZURE_VISION_ADDRESS, and AZURE_LANGUAGE_ADDRESS can be found in the "Keys and Endpoint" tab of the Azure AI Foundry service, under the "AI Services" subtab at the bottom of the page. "/contentsafety/text:analyze?api-version=2024-09-01" should be appended to AZURE_MODERATION_ADDRESS.

- The default container allocation of 0.5 CPU and 1G of RAM (container = replica here) allows max 2 users per replica, since one ChatSafetyAI session consumes approx 400MB of RAM. The default consumption-based workload profile has a hard limit of 4 CPUs and 8 GBs of RAM, so it allows 4/0.5 = 8 replicas to be started, CPU being the limiting factor here (hence the --max-replicas 8 below). This setup supports up to 8 × 2 = 16 concurrent users. If more is needed, switch to a general-purpose workload profile: https://learn.microsoft.com/en-us/azure/container-apps/workload-profiles-overview

```
az containerapp update \
  --name csai-mre-chatbot \
  --resource-group $RESOURCE_GROUP \
  --max-replicas 8 \
  --scale-rule-name http-scaler \
  --scale-rule-type http \
  --scale-rule-metadata concurrentRequests=1 \
  --set-env-vars \
    IS_AZURE=TRUE \
    IS_LOCAL=FALSE \
    CHECK_LOCK_FILE=FALSE \
    MAX_INACTIVE_TIME=24 \
    AZURE_STORAGE_ACCOUNT=$STORAGE_ACCOUNT \
    AZURE_STORAGE_ACCOUNT_CONTAINER_NAME=database \
    AZURE_STORAGE_ACCOUNT_KEY='<your-storage-account-key>' \
    UTILITIES_ADDRESS="https://csai-mre-utilities.$ENV_UNIQUE.$LOCATION.azurecontainerapps.io/" \
    PREDICTIONS_ADDRESS="https://csai-mre-prediction.$ENV_UNIQUE.$LOCATION.azurecontainerapps.io/" \
    NLP_ADDRESS="https://csai-mre-attribute-detection.$ENV_UNIQUE.$LOCATION.azurecontainerapps.io/" \
    OPENAI_MODEL_NAME_GPT4=gpt-4.1-2025-04-14 \
    OPENAI_MODEL_NAME_GPT35=gpt-4.1-mini-2025-04-14 \
    AZURE_OPENAI_ADDRESS_GPT4="https://$AISERVICES_NAME.cognitiveservices.azure.com/openai/deployments/gpt-4.1/chat/completions?api-version=2025-01-01-preview" \
    AZURE_OPENAI_ADDRESS_GPT35="https://$AISERVICES_NAME.cognitiveservices.azure.com/openai/deployments/gpt-4.1-mini/chat/completions?api-version=2025-01-01-preview" \
    AZURE_MODERATION_ADDRESS="https://$AISERVICES_NAME.services.ai.azure.com/contentsafety/text:analyze?api-version=2024-09-01" \
    AZURE_VISION_ADDRESS="https://$AISERVICES_NAME.services.ai.azure.com/" \
    AZURE_LANGUAGE_ADDRESS="https://$AISERVICES_NAME.services.ai.azure.com/" \
    USE_MANAGED_IDENTITY=TRUE
```

##### c. Dashboard

```
az containerapp update \
  --name csai-mre-dashboard \
  --resource-group $RESOURCE_GROUP \
  --set-env-vars \
    AZURE_COMPANY_NAME=$COMPANY_NAME \
    AZURE_STORAGE_ACCOUNT=$STORAGE_ACCOUNT \
    AZURE_STORAGE_ACCOUNT_CONTAINER_NAME=database \
    IS_AZURE=TRUE \
    IS_LOCAL=FALSE \
    USE_MANAGED_IDENTITY=TRUE
```

#### 9) Set up Azure Entra ID for the two user-facing apps

##### a. Register apps in Entra ID, create service principals, and enable authentication

```
for app in csai-mre-chatbot csai-mre-dashboard; do
  # Register the app and get appId
  appId=$(az ad app create \
    --display-name $app \
    --web-redirect-uris "https://$app.$ENV_UNIQUE.$LOCATION.azurecontainerapps.io/.auth/login/aad/callback" \
    --sign-in-audience AzureADandPersonalMicrosoftAccount \
    --requested-access-token-version 2 \
    --query appId -o tsv)

  # Create the service principal
  az ad sp create --id $appId

  # Enable authentication on the Container App
  az containerapp auth update \
    --name $app \
    --resource-group $RESOURCE_GROUP \
    --enabled true \
    --unauthenticated-client-action RedirectToLoginPage
done
```

##### b. In the Azure Portal, for each app:
- Go to "Add identity provider" in the "Authentication" tab, and set:
  - Provider: Microsoft
  - Tenant type: Workforce
  - App registration type: Pick an existing app registration
  - Set Client secret expiration to 24 months
  - Issuer URL: https://login.microsoftonline.com/<tenant-id>/v2.0 (use the URL printed below, same for both apps)

```
TENANT_ID=$(az account show --query tenantId --output tsv)
echo "Issuer URL: https://login.microsoftonline.com/$TENANT_ID/v2.0"
```

##### c. In the Azure Portal:

- In Entra ID App, click "Applications", "All applications", and for each app:
- In the "Authentication" tab, "Settings" tab, ensure ID tokens is checked.

#### 10) Assign managed identity roles 

Helper shell function
```
assign_role_to_containerapp_identity() {
  local containerapp_name="$1"
  local resource_group="$2"
  local target_resource_name="$3"
  local target_resource_type="$4"   # e.g., 'storage', 'keyvault', 'cognitiveservices'
  local role_name="$5"

  echo "🔍 Resolving principal ID for $containerapp_name..."
  principal_id=$(az containerapp show \
    --name "$containerapp_name" \
    --resource-group "$resource_group" \
    --query identity.principalId \
    --output tsv)

  echo "🔍 Resolving resource ID for $target_resource_name ($target_resource_type)..."
  case "$target_resource_type" in
    storage)
      scope=$(az storage account show \
        --name "$target_resource_name" \
        --resource-group "$resource_group" \
        --query id \
        --output tsv)
      ;;
    keyvault)
      scope=$(az keyvault show \
        --name "$target_resource_name" \
        --resource-group "$resource_group" \
        --query id \
        --output tsv)
      ;;
    cognitiveservices)
      scope=$(az cognitiveservices account show \
        --name "$target_resource_name" \
        --resource-group "$resource_group" \
        --query id \
        --output tsv)
      ;;
    *)
      echo "❌ Unsupported resource type: $target_resource_type"
      return 1
      ;;
  esac

  echo "🔐 Assigning role '$role_name' to identity..."
  if az role assignment list \
    --assignee "$principal_id" \
    --scope "$scope" \
    --role "$role_name" \
    --query "[].id" \
    --output tsv | grep -q .
  then
    echo "✅ Role already assigned — skipping"
  else
    az role assignment create \
      --assignee-object-id "$principal_id" \
      --assignee-principal-type ServicePrincipal \
      --role "$role_name" \
      --scope "$scope"
  fi
}
```

Format: `app_name resource_group resource_name resource_type role_name`
```
assignments=(
  # ChatSafetyAI (chatbot)
  "csai-mre-chatbot $RESOURCE_GROUP $STORAGE_ACCOUNT storage 'Storage Blob Data Contributor'"
  "csai-mre-chatbot $RESOURCE_GROUP $AISERVICES_NAME cognitiveservices 'Cognitive Services OpenAI User'"
  "csai-mre-chatbot $RESOURCE_GROUP $AISERVICES_NAME cognitiveservices 'Cognitive Services User'"
  # Dashboard
  "csai-mre-dashboard $RESOURCE_GROUP $STORAGE_ACCOUNT storage 'Storage Blob Data Contributor'"
  # Utilities
  "csai-mre-utilities $RESOURCE_GROUP $STORAGE_ACCOUNT storage 'Storage Blob Data Contributor'"
  "csai-mre-utilities $RESOURCE_GROUP $AISERVICES_NAME cognitiveservices 'Cognitive Services OpenAI User'"
)

for assignment in "${assignments[@]}"; do
  read -r app resource_group resource_name resource_type role_name <<< "$assignment"
  # Remove surrounding single quotes from role_name
  role_name=${role_name//\'/}
  assign_role_to_containerapp_identity "$app" "$resource_group" "$resource_name" "$resource_type" "$role_name"
done
```

Verify Configuration manually in the Azure AI Portal. On the Container:
- Go to Portal > Container App > Identity
- Ensure System-assigned is On and principal ID is shown
- On the Target Resource:
- Go to Access Control (IAM) > Role assignments
