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
COMPANY_NAME="CompanyTest"                       # Company name for dashboard
APP_SERVICE_PLAN="csai-mre-plan"                 # For APIs
SEARCH_SERVICE_NAME="csai-search-mre"            # For Search Service
SEARCH_INDEX="rag-index"                         # ..
SEARCH_INDEXER="rag-indexer"                     # ..
SEARCH_DATASOURCE="blob-datasource"              # ..
SEARCH_SKILLSET="rag-skillset"                   # ..
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

#### 3) Create Storage Account.
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
In the "Properties" tab of the landing page of the service, leave everything default except:
- Blob soft delete (enable it with the duration of your choice everywhere, e.g. 30 days), and make sure versioning and point in time restore are disabled.
- Set minimum TLS version to 1.2
- In "Containers", add a container named "database" (3 directories will automatically be created inside it by the chatbot: "ChatSafetyAI_user_data", "custom_databases", and "session_logs")


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
  - gpt-5.1-2025-11-13
  - gpt-4.1-2025-04-14
  - gpt-4.1-mini-2025-04-14
  - gpt-4o-transcribe
  - text-embedding-3-large
  - NOTE: due to region availability, Azure may silently deploy one or more models in a different resource than `AISERVICES_NAME`. Save the names of the new resources created by Azure and pass them as `OUTSIDE_RESOURCES` at the beginning of Step 9.
  - NOTE: gpt-5.1 may require filing an access request with Microsoft and waiting a few days until it gets approved 
  - NOTE: in "Customize", for all models:
    - Opt out of automatic model version upgrades !!!
    - Check quotas! If less than 1k requests per minute (RPM) and 1M tokens per minute (TPM), request increase here: https://aka.ms/oai/quotaincrease

- c) For each resource associated with model deployments: In "Guardrails + controls", create a new content filter with all thresholds set to minimum, and jailbreak and indirect attacks set to "annotate only". Do that for input and output, and apply filter to all deployments.

## SafetyAI-Managed Services

#### 5) Upload Docker images to container registry
```
# Automatically retrieve ACR credentials and log in
ACR_USERNAME=$(az acr credential show --name $ACR_NAME --query username --output tsv)
ACR_PASSWORD=$(az acr credential show --name $ACR_NAME --query passwords[0].value --output tsv)

# We use the retrieved credentials to log docker in with sudo permissions
sudo docker login $ACR_NAME.azurecr.io --username $ACR_USERNAME --password $ACR_PASSWORD
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

# VERIFY they are not empty
if [[ -z "$WORKSPACE_ID" || -z "$WORKSPACE_KEY" ]]; then
  echo "❌ ERROR: Failed to retrieve Workspace ID or Key."
  echo "Check if workspace '$LOG_WORKSPACE' exists in group '$RESOURCE_GROUP'."
else
  echo "✅ Success! ID: $WORKSPACE_ID"
  echo "✅ Key found (hidden)"
fi

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

##### d. Retrieve the new Environment Unique String
```
ACA_SUFFIX=$(az containerapp env show \
  --name $CONTAINERAPPS_ENV \
  --resource-group $RESOURCE_GROUP \
  --query properties.defaultDomain \
  --output tsv)

echo "✅ Environment Domain: $ACA_SUFFIX"
```

#### 7) Create services

##### Find a region that will allow an App Service Plan
```
# ==============================================================================
# SAFE REGION PROSPECTOR (Finds a region with B-series Quota)
# ==============================================================================
# Regions to test (prioritizing those close to East US)
REGIONS=("eastus2" "northcentralus" "canadacentral" "westus3" "westcentralus" "francecentral")

TEMP_RG="rg-quota-finder-temp"
TEST_PLAN="quota-test-plan"

echo "--- Starting Safe Region Prospector ---"

for REGION in "${REGIONS[@]}"; do
    echo ""
    echo "Testing Region: $REGION..."
    
    # 1. Create a temporary RG (Fast & Free)
    az group create --name "$TEMP_RG" --location "$REGION" --output none
    
    # 2. Try to create a B1 Plan (Cheapest valid tier)
    az appservice plan create \
        --name "$TEST_PLAN" \
        --resource-group "$TEMP_RG" \
        --location "$REGION" \
        --sku B1 \
        --is-linux > /dev/null 2>&1
    
    # 3. Check Result
    if [ $? -eq 0 ]; then
        echo "✅ SUCCESS! Region '$REGION' allowed creating a B1 plan."
        echo "-------------------------------------------------------"
        echo ">>> WINNING REGION: $REGION"
        echo "-------------------------------------------------------"
        
        # Cleanup
        az group delete --name "$TEMP_RG" --yes --no-wait
        
        # EXPORT THE VARIABLE SO WE CAN USE IT
        export APP_LOCATION=$REGION
        break
    else
        echo "❌ Failed in $REGION (Quota Blocked)."
        az group delete --name "$TEMP_RG" --yes --no-wait
    fi
done

if [ -z "$APP_LOCATION" ]; then
    echo "☹️ Bad news: All tested regions are blocked. You may need to file a support ticket."
else
    echo "🎉 Ready to proceed! APP_LOCATION is set to: $APP_LOCATION"
fi
```

##### a. Create App Service Plan for APIs
```
az appservice plan create \
  --name $APP_SERVICE_PLAN \
  --resource-group $RESOURCE_GROUP \
  --location $APP_LOCATION \
  --sku B3 \
  --is-linux
```

##### b. Create API Web Apps
```
# Format: "App-Name Image-Name Port"
api_configs=(
  "csai-mre-prediction safetyai_prediction 4387"
  "csai-mre-attribute-detection safetyai_attribute_detection 4388"
  "csai-mre-utilities chatsafetyai_utilities 4386"
)

for config in "${api_configs[@]}"; do
  read -r app_name image_name port <<< "$config"
  
  echo "Deploying Web App: $app_name..."
  
  # Create Web App (automatically goes to West US 3 because of the Plan)
  az webapp create \
    --name "$app_name" \
    --resource-group "$RESOURCE_GROUP" \
    --plan "$APP_SERVICE_PLAN" \
    --deployment-container-image-name "$ACR_NAME.azurecr.io/$image_name:latest" \
    --assign-identity "[system]"

  # Configure Port & Registry Access
  az webapp config appsettings set \
    --name "$app_name" \
    --resource-group "$RESOURCE_GROUP" \
    --settings \
      WEBSITES_PORT=$port \
      WEBSITES_CONTAINER_START_TIME_LIMIT=600 \
      DOCKER_REGISTRY_SERVER_URL="https://$ACR_NAME.azurecr.io"
done
```

##### c. Create Container Apps (Chatbot & Dashboard)
```
for app in \
  "csai-mre-chatbot chatsafetyai 3838" \
  "csai-mre-dashboard dashboard 3838"
do
  read -r app_name image_name port <<< "$app"
  
  az containerapp create \
    --name $app_name \
    --resource-group $RESOURCE_GROUP \
    --environment $CONTAINERAPPS_ENV \
    --image "$ACR_NAME.azurecr.io/$image_name:latest" \
    --target-port $port \
    --ingress external \
    --registry-server "$ACR_NAME.azurecr.io" \
    --registry-identity system \
    --min-replicas 1 \
    --system-assigned
done
```

#### 8) Create Search Service

```
# 1. Make the script executable
chmod +x configure_azure_search.sh

# 2. Get AI Services Subdomain
# This returns the full URL, e.g., "https://csai-mre-aiservices.cognitiveservices.azure.com/"
FULL_ENDPOINT=$(az cognitiveservices account show \
  --name $AISERVICES_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "properties.endpoint" \
  --output tsv)

# Clean the URL to get just the domain (remove 'https://' and trailing '/')
# Result: "csai-mre-aiservices.cognitiveservices.azure.com"
CLEAN_DOMAIN=$(echo "$FULL_ENDPOINT" | sed -e 's|^https://||' -e 's|/$||')

echo "✅ Detected Domain: $CLEAN_DOMAIN"

# We use this same valid domain for both OpenAI and General Skills
AOAI_SUBDOMAIN="$CLEAN_DOMAIN"
AISERVICES_SUBDOMAIN="$CLEAN_DOMAIN"

# 3. Run the Configuration Script (requires a few consecutive runs, waiting a few mins between each run, for everything to execute properly)
# Order: Tenant, Sub, RG, Loc, Storage, Container, AISvc, AOAI_Sub, AI_Sub, SearchName, FuncName, VirtDir, DS, Idx, Skill, Idxr, ContainerAppName, RbacMode
./configure_azure_search.sh \
  "$TENANT_ID" \
  "$SUBSCRIPTION_ID" \
  "$RESOURCE_GROUP" \
  "$LOCATION" \
  "$STORAGE_ACCOUNT" \
  "database" \
  "$AISERVICES_NAME" \
  "$AOAI_SUBDOMAIN" \
  "$AISERVICES_SUBDOMAIN" \
  "$SEARCH_SERVICE_NAME" \
  "func-placeholder" \
  "custom_databases" \
  "$SEARCH_DATASOURCE" \
  "$SEARCH_INDEX" \
  "$SEARCH_SKILLSET" \
  "$SEARCH_INDEXER" \
  "csai-mre-chatbot" \
  "false" # not possible to disable keys programmatically, do it manually in the Azure Portal if needed


# ==================
# CLEAN UP TMP FILES
# ==================

# Cleanup temporary JSON files
rm index.json index.json.bak \
   skillset.json skillset.json.bak \
   indexer.json indexer.json.bak \
   2>/dev/null


# =====================
# VERIFICATION COMMANDS
# =====================

echo "🔄 1. Resetting Indexer (Clearing history)..."
az rest --method post \
  --resource https://search.azure.com \
  --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexers/${SEARCH_INDEXER}/reset?api-version=2024-07-01"

echo "▶️ 2. Running Indexer..."
az rest --method post \
  --resource https://search.azure.com \
  --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexers/${SEARCH_INDEXER}/run?api-version=2024-07-01"

# wait a couple of minutes...

echo "📊 3. Checking Execution Status..."
STATUS=$(az rest --method get \
  --resource https://search.azure.com \
  --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexers/${SEARCH_INDEXER}/status?api-version=2024-07-01" \
  --query "lastResult")

echo "$STATUS"
# Look for: "itemsProcessed" > 0 and "status": "success"

echo "🔎 4. Final Proof: Inspecting 1 Document..."
# This confirms your custom fields (user_id, user_folder) and vector embeddings exist.
az rest --method post \
   --resource https://search.azure.com \
   --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexes/${SEARCH_INDEX}/docs/search?api-version=2024-07-01" \
   --headers '{"Content-Type": "application/json"}' \
   --body '{"search": "*", "top": 1, "select": "file_name, user_id, user_folder, chunk"}'

```

#### 9) Set arguments and environment variables needed by each service

```
MAIN_ENDPOINT=$(az cognitiveservices account show \
  --name "$AISERVICES_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query properties.endpoint \
  --output tsv)

# --- DEFINE OUTSIDE RESOURCES ---
# List all your outside resources inside the parentheses, separated by spaces.
# Example: OUTSIDE_RESOURCES=('resource1-name' 'resource2-name')
# If all models were deployed in AISERVICES_NAME, leave it empty: OUTSIDE_RESOURCES=()
OUTSIDE_RESOURCES=('tixie-ml1ae2pw-eastus2')

# Verify they all exist
for resource in "${OUTSIDE_RESOURCES[@]}"; do
    echo "🔍 Verifying existence of Outside Resource: '$resource'..."
    check_rg=$(az cognitiveservices account list --query "[?name=='$resource'].resourceGroup" --output tsv)
    
    if [ -z "$check_rg" ]; then
        echo "❌ ERROR: Could not find resource '$resource'. Please check the name."
        exit 1
    else
        echo "   ✅ Found in group: $check_rg"
    fi
done
```

##### a. Utilities API

```
HOST_UTILITIES=$(az webapp show \
  --name csai-mre-utilities \
  --resource-group $RESOURCE_GROUP \
  --query defaultHostName \
  --output tsv)

az webapp config appsettings set \
  --name csai-mre-utilities \
  --resource-group $RESOURCE_GROUP \
  --settings \
    AZURE_STORAGE_ACCOUNT=$STORAGE_ACCOUNT \
    AZURE_STORAGE_ACCOUNT_CONTAINER_NAME=database \
    USE_MANAGED_IDENTITY=TRUE \
    IS_AZURE=TRUE \
    UTILITIES_ADDRESS="https://$HOST_UTILITIES/"
```

##### b. ChatSafetyAI

###### NOTES:

- **Session Affinity**: Enable session affinity in the app's page on the Azure Portal, under "Ingress".

- **API Addresses (Dynamic Fetching)**:
  - `UTILITIES_ADDRESS`, `PREDICTIONS_ADDRESS`, and `NLP_ADDRESS` can be found in the respective overview pages of the apps in the Azure Portal ("Application Url") - they must end with a "/".
  - The commands below automatically fetch these addresses using `az webapp show` (since they are all Web Apps), no action needed.

- **Azure OpenAI Endpoints**:
  - `AZURE_OPENAI_ADDRESS_GPT4`, `AZURE_OPENAI_ADDRESS_GPT35` (actually just a smaller version of GPT-4.1), `AZURE_OPENAI_RESPONSES_ADDRESS`, `AZURE_OPENAI_ADDRESS_THINKING`, and `AZURE_OPENAI_ADDRESS_AUDIO` can be found in the "Endpoint" area in the Azure AI Foundry page of the corresponding deployment.
  - They are of the form:
    - `AZURE_OPENAI_ADDRESS_GPT4`: `{YOUR_ENDPOINT}openai/deployments/gpt-4.1/chat/completions?api-version=2025-01-01-preview`
    - `AZURE_OPENAI_ADDRESS_GPT35`: `{YOUR_ENDPOINT}openai/deployments/gpt-4.1-mini/chat/completions?api-version=2025-01-01-preview`
    - `AZURE_OPENAI_RESPONSES_ADDRESS`: `{YOUR_ENDPOINT}openai/responses?api-version=2025-04-01-preview`
    - `AZURE_OPENAI_ADDRESS_THINKING`: should be the same as `AZURE_OPENAI_RESPONSES_ADDRESS` as Azure exposes GPT-5.1 by default through the new Responses API
    - `AZURE_OPENAI_ADDRESS_AUDIO`: `{YOUR_ENDPOINT}/openai/deployments/gpt-4o-transcribe/audio/transcriptions?api-version=2025-03-01-preview`
  - **Note:** `YOUR_ENDPOINT` can vary between the endpoint of the main resource or that of one or more new resources silently created by Azure in case of region unavailability; you should manually fetch all addresses from AI Foundry and paste them below.
- **AI Services Endpoints**:
  - `AZURE_MODERATION_ADDRESS`, `AZURE_VISION_ADDRESS`, and `AZURE_LANGUAGE_ADDRESS` can be found in the "Keys and Endpoint" tab of the Azure AI Foundry service, under the "AI Services" subtab at the bottom of the page.
  - They are the same endpoint (`MAIN_ENDPOINT` as defined above), except that `/contentsafety/text:analyze?api-version=2024-09-01` should be appended to `AZURE_MODERATION_ADDRESS`.

- **Resource Limits**:
  - The default container allocation of 0.5 CPU and 1G of RAM (container = replica here) allows max 2 users per replica, since one ChatSafetyAI session consumes approx 400MB of RAM.
  - The default consumption-based workload profile has a hard limit of 4 CPUs and 8 GBs of RAM, so it allows 4/0.5 = 8 replicas to be started, CPU being the limiting factor here (hence the `--max-replicas 8` below). This setup supports up to 8 × 2 = 16 concurrent users. If more is needed, switch to a [general-purpose workload profile](https://learn.microsoft.com/en-us/azure/container-apps/workload-profiles-overview).

```
# --- Dynamic URL Fetching for APIs (All are Web Apps) ---

# 1. Fetch Web App Hostname (Utilities)
HOST_UTILITIES=$(az webapp show --name csai-mre-utilities --resource-group $RESOURCE_GROUP --query defaultHostName --output tsv)
UTILITIES_ADDRESS="https://$HOST_UTILITIES/"

# 2. Fetch Web App Hostname (Prediction)
HOST_PREDICTION=$(az webapp show --name csai-mre-prediction --resource-group $RESOURCE_GROUP --query defaultHostName --output tsv)
PREDICTIONS_ADDRESS="https://$HOST_PREDICTION/"

# 3. Fetch Web App Hostname (Attribute Detection / NLP)
HOST_NLP=$(az webapp show --name csai-mre-attribute-detection --resource-group $RESOURCE_GROUP --query defaultHostName --output tsv)
NLP_ADDRESS="https://$HOST_NLP/"

az containerapp update \
  --name csai-mre-chatbot \
  --resource-group $RESOURCE_GROUP \
  --max-replicas 8 \
  --scale-rule-name http-scaler \
  --scale-rule-type http \
  --scale-rule-metadata concurrentRequests=1 \
  --set-env-vars \
    IS_AZURE=TRUE \
    UTILITIES_ADDRESS="$UTILITIES_ADDRESS" \
    PREDICTIONS_ADDRESS="$PREDICTIONS_ADDRESS" \
    NLP_ADDRESS="$NLP_ADDRESS" \
    OPENAI_MODEL_NAME_GPT4=gpt-4.1-2025-04-14 \
    OPENAI_MODEL_NAME_GPT35=gpt-4.1-mini-2025-04-14 \
    CHECK_LOCK_FILE=TRUE \
    MAX_INACTIVE_TIME=24 \
    DISABLE_KEEPALIVE=FALSE \
    RECORD_LOGS=TRUE \
    CHECK_USER_BUDGETS=FALSE \
    SHOW_CUSTOM_DB=TRUE \
    SEARCH_ENDPOINT="https://$SEARCH_SERVICE_NAME.search.windows.net" \
    SEARCH_INDEXER="rag-indexer" \
    SEARCH_INDEX="rag-index" \
    AZURE_STORAGE_ACCOUNT_CONTAINER_NAME=database \
    AZURE_STORAGE_ACCOUNT=$STORAGE_ACCOUNT \
    OPENAI_MODEL_NAME_THINKING=gpt-5.1-2025-11-13 \
    MAX_RECORDING_TIME_MINS=5 \
    IS_LOCAL=FALSE \
    AZURE_MODERATION_ADDRESS="${MAIN_ENDPOINT}contentsafety/text:analyze?api-version=2024-09-01" \
    AZURE_VISION_ADDRESS="${MAIN_ENDPOINT}" \
    AZURE_LANGUAGE_ADDRESS="${MAIN_ENDPOINT}" \
    USE_MANAGED_IDENTITY=TRUE \
    AZURE_OPENAI_RESPONSES_ADDRESS=https://csai-aiservices-mre.cognitiveservices.azure.com/openai/responses?api-version=2025-04-01-preview\
    AZURE_OPENAI_ADDRESS_THINKING=https://csai-aiservices-mre.cognitiveservices.azure.com/openai/responses?api-version=2025-04-01-preview \
    AZURE_OPENAI_ADDRESS_GPT4=https://csai-aiservices-mre.cognitiveservices.azure.com/openai/deployments/gpt-4.1/chat/completions?api-version=2025-01-01-preview \
    AZURE_OPENAI_ADDRESS_GPT35=https://csai-aiservices-mre.cognitiveservices.azure.com/openai/deployments/gpt-4.1-mini/chat/completions?api-version=2025-01-01-preview \
    AZURE_OPENAI_ADDRESS_AUDIO=https://tixie-ml1ae2pw-eastus2.cognitiveservices.azure.com/openai/deployments/gpt-4o-transcribe/audio/transcriptions?api-version=2025-03-01-preview
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

#### 10) Assign managed identity roles 

```
# Helper shell function
assign_role_to_containerapp_identity() {
  local containerapp_name="$1"
  local target_resource_group="$2"  # Renamed for clarity: matches the Target Resource
  local target_resource_name="$3"
  local target_resource_type="$4"   # e.g., 'storage', 'search', 'cognitiveservices'
  local role_name="$5"

  echo "🔍 Resolving principal ID for $containerapp_name..."
  # NOTE: We use the global $RESOURCE_GROUP for the app identity, as apps are always in the main group.
  # We try Container App first, then fall back to Web App (for Utilities API).
  principal_id=$(az containerapp show \
    --name "$containerapp_name" \
    --resource-group "$RESOURCE_GROUP" \
    --query identity.principalId \
    --output tsv 2>/dev/null)

  if [ -z "$principal_id" ]; then
      echo "   ...not found as Container App, checking Web App..."
      principal_id=$(az webapp identity show \
        --name "$containerapp_name" \
        --resource-group "$RESOURCE_GROUP" \
        --query principalId \
        --output tsv 2>/dev/null)
  fi

  if [ -z "$principal_id" ]; then
      echo "❌ ERROR: Could not find identity for $containerapp_name. Check if the app exists."
      return 1
  fi

  echo "🔍 Resolving resource ID for $target_resource_name ($target_resource_type) in $target_resource_group..."
  case "$target_resource_type" in
    storage)
      scope=$(az storage account show \
        --name "$target_resource_name" \
        --resource-group "$target_resource_group" \
        --query id \
        --output tsv)
      ;;
    keyvault)
      scope=$(az keyvault show \
        --name "$target_resource_name" \
        --resource-group "$target_resource_group" \
        --query id \
        --output tsv)
      ;;
    cognitiveservices)
      scope=$(az cognitiveservices account show \
        --name "$target_resource_name" \
        --resource-group "$target_resource_group" \
        --query id \
        --output tsv)
      ;;
    search)
      scope=$(az search service show \
        --name "$target_resource_name" \
        --resource-group "$target_resource_group" \
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

# Format: `app_name target_resource_group target_resource_name target_resource_type role_name`
assignments=(
  # ChatSafetyAI (chatbot)
  "csai-mre-chatbot $RESOURCE_GROUP $STORAGE_ACCOUNT storage 'Storage Blob Data Contributor'"
  "csai-mre-chatbot $RESOURCE_GROUP $AISERVICES_NAME cognitiveservices 'Cognitive Services OpenAI User'"
  "csai-mre-chatbot $RESOURCE_GROUP $AISERVICES_NAME cognitiveservices 'Cognitive Services User'"
  "csai-mre-chatbot $RESOURCE_GROUP $SEARCH_SERVICE_NAME search 'Search Index Data Contributor'"
  
  # Dashboard
  "csai-mre-dashboard $RESOURCE_GROUP $STORAGE_ACCOUNT storage 'Storage Blob Data Contributor'"
  
  # Utilities
  "csai-mre-utilities $RESOURCE_GROUP $STORAGE_ACCOUNT storage 'Storage Blob Data Contributor'"
  "csai-mre-utilities $RESOURCE_GROUP $AISERVICES_NAME cognitiveservices 'Cognitive Services OpenAI User'"
)

# --- Apply Standard Assignments ---
echo "=== Applying Standard Role Assignments ==="
for assignment in "${assignments[@]}"; do
  read -r app target_rg resource_name resource_type role_name <<< "$assignment"
  # Remove surrounding single quotes from role_name
  role_name=${role_name//\'/}
  assign_role_to_containerapp_identity "$app" "$target_rg" "$resource_name" "$resource_type" "$role_name"
done

# --- [CONDITIONAL] Apply Outside Resource Permissions ---
# Iterates through the OUTSIDE_RESOURCES array defined in Step 9
if [ ${#OUTSIDE_RESOURCES[@]} -gt 0 ]; then
    echo "=== 🌍 Applying Permissions for Outside Resources ==="
    
    for resource_name in "${OUTSIDE_RESOURCES[@]}"; do
        # 1. Dynamically find the Resource Group for this specific outside resource
        target_rg=$(az cognitiveservices account list --query "[?name=='$resource_name'].resourceGroup" --output tsv)
        
        if [ -n "$target_rg" ]; then
            echo " > Granting Chatbot access to: $resource_name (RG: $target_rg)..."
            # 2. Call helper function with the specific RG found
            assign_role_to_containerapp_identity \
                "csai-mre-chatbot" \
                "$target_rg" \
                "$resource_name" \
                "cognitiveservices" \
                "Cognitive Services OpenAI User"
        else
            echo "⚠️  WARNING: Could not find Resource Group for '$resource_name'. Skipping permission assignment."
        fi
    done
else
    echo "=== 🏠 Skipping Outside Permissions (List is empty) ==="
fi
```

Verify Configuration manually in the Azure AI Portal. On the Container:
 - Go to Portal > Container App > Identity
 - Ensure System-assigned is On and principal ID is shown
 - On the Target Resource (Search, AI, Storage):
 - Go to Access Control (IAM) > Role assignments


#### 11) Set up Azure Entra ID for the two user-facing apps

##### a. Register apps in Entra ID, create service principals

```
# Loop for both apps
for app in csai-mre-chatbot csai-mre-dashboard; do
  echo "Configuring Entra ID for $app..."

  # 1. Get the proper FQDN (URL) of the Container App
  FQDN=$(az containerapp show --name $app --resource-group $RESOURCE_GROUP --query properties.configuration.ingress.fqdn -o tsv)
  REDIRECT_URI="https://$FQDN/.auth/login/aad/callback"
  echo "   Redirect URI: $REDIRECT_URI"

  # 2. Register the App in Entra ID
  # We use AzureADMyOrg (Single Tenant) which matches the "Workforce" default
  appId=$(az ad app create \
    --display-name "$app" \
    --web-redirect-uris "$REDIRECT_URI" \
    --sign-in-audience AzureADMyOrg \
    --query appId -o tsv)

  # 3. Enable ID Tokens (Automates Step 11c)
  az ad app update --id $appId --enable-id-token-issuance true

  # 4. Create Service Principal
  az ad sp create --id $appId --output none

  echo "   ✅ Registered. App ID: $appId"
done
```

##### b. Configure Authentication in the Azure Portal (Manual)

Perform this step manually for **both** `csai-mre-chatbot` and `csai-mre-dashboard`.

**Phase 1: Generate the Secret**
1.  In the Azure Portal, search for and open **App registrations**.
2.  Click the **All applications** tab and select your app (e.g., `csai-mre-chatbot`).
3.  **Copy the "Application (client) ID"** from the Overview blade.
4.  In the left menu, select **Certificates & secrets**.
5.  Click **+ New client secret**.
    * *Description:* `AuthKey`
    * *Expires:* **24 months** (730 days).
    * Click **Add**.
6.  **Important:** Immediately copy the **Value** (not the Secret ID).

**Phase 2: Link to Container App**
1.  Navigate to your Container App resource (e.g., `csai-mre-chatbot`).
2.  In the left menu, select **Authentication**.
3.  Click **Add identity provider**.
4.  Select **Microsoft**.
5.  **App registration type**: Select **Provide the details of an existing app registration**.
    * *Note: Do not use "Pick an existing..." as it prevents manual secret management.*
6.  **Application (client) ID**: Paste the ID you copied in Phase 1.
7.  **Client secret (value)**: Paste the Secret Value you copied in Phase 1.
8.  **Issuer URL**:
    * If the Portal auto-fills this (e.g., with `https://sts.windows.net/...`), **leave it as is**.
    * If it is empty, use: `https://login.microsoftonline.com/<TENANT_ID>/v2.0` (replace `<TENANT_ID>` with your actual Tenant ID).
9.  Click **Add**.

*Repeat Phase 1 & 2 for `csai-mre-dashboard`.*
