# Azure AI Search Configuration Script

This folder contains the orchestration script for the Search Service indexing pipeline.

## 📁 File: `configure_azure_search_service_enterprise.sh`

### 🏗️ Responsibility Boundary: "Plumbing vs. Wiring"
To maintain a clean separation of concerns, the deployment is split as follows:

1. **Infrastructure:** Creates the Search Service, Storage, and AI Services. Handles "outbound" Managed Identity roles.
2. **This Script:** Defines the Index schema, Skillsets, and Indexers. Handles "inbound" RBAC for the Container App and the Script Executor.

---

### 📋 Prerequisites
For this script to succeed, the infrastructure must ensure:

| Feature | Requirement |
| :--- | :--- |
| **Identity** | Search Service must have a `SystemAssigned` Managed Identity. |
| **Auth Mode** | The script supports both Hybrid (`aadOrApiKey`) and strict RBAC (`disableLocalAuth: true`). |
| **Outbound RBAC** | Search MI needs `Storage Blob Data Reader`, `Cognitive Services User`, and `Cognitive Services OpenAI User` on target resources. |

---

### ✅ Pre-Flight Environment Verification Checklist
Before running the configuration script in a new environment (especially PROD), verify the following Identity and Networking configurations directly in the Azure Portal.

#### 1. IAM & Role Assignments (The "Lock & Key" Check)
* **STEP 0: Search Service Identity Check:**
  * Navigate to **Azure AI Search Service** -> **Identity** (under Settings).
  * **Requirement:** Under the **System assigned** tab, the Status toggle MUST be **"On"**, and an Object (principal) ID must be visible. (If it is off, turn it on and save).
* **Pipeline Execution Identity (The Key Fetcher):**
  * Navigate to **Azure AI Services** -> **Access control (IAM)** -> **Role assignments**. 
  * Search for the Service Principal/User running the CI/CD pipeline.
  * **Requirement:** Must have **Cognitive Services Contributor** (Allows the script to fetch the API key).
* **Search Service as Document Reader:**
  * Navigate to the **Storage Account** -> **Access control (IAM)** -> **Role assignments**. 
  * Search for the Azure AI Search Service name.
  * **Requirement:** Must have **Storage Blob Data Reader** (or Contributor).
* **Search Service as Embedder:**
  * Navigate to **Azure AI Services** -> **Access control (IAM)** -> **Role assignments**. 
  * Search for the Azure AI Search Service name.
  * **Requirement:** Must have **Cognitive Services OpenAI User**.

#### 2. Networking & Shared Private Links
* Navigate to **Azure AI Search** -> **Networking** -> **Shared private access**.
* Verify the three required links (`blob`, `cognitiveservices`, `openai`) exist.
* **Requirement:** Their status must explicitly say **"Approved"** (not "Pending"). If pending, navigate to the target Storage/AI resources and approve them immediately.

---

### 🔒 Secure Environments (Shared Private Links)
If the target environment has **Public Network Access disabled** on the Storage or AI resources, you must provision **Shared Private Link Resources** originating from the Search Service so it can securely access the data.

**Required Shared Private Links (`groupId`):**
1. Blob Storage (`blob`)
2. AI Services / Vision (`cognitiveservices`)
3. AI Services / OpenAI (`openai` or `openai_account`)

#### Option A: Manual Workflow via Azure Portal (Current)
Deploying a Shared Private Link puts the connection into a `Pending` state. **These endpoints must be manually approved in the Azure Portal before this configuration script runs;** otherwise, skillset validation will fail with a `403 Forbidden`.

1. **Request Links:** In the Azure Portal, go to your Search Service -> **Networking** -> **Shared private access** and add the 3 links above.
2. **Approve Storage:** Go to your Storage Account -> **Networking** -> **Private endpoint connections** -> Check the pending request and click **Approve**.
3. **Approve AI Services:** Go to your AI Multi-Service Account -> **Networking** -> **Private endpoint connections** -> Check the pending requests and click **Approve**.

Once all show as **Approved** in the Search Service, you may run this script.

#### Option B: Automated Workflow via Bicep & Pipeline (Future Reference)
To fully automate this in the future, add the links to the Bicep template and add an approval task to the CI/CD pipeline immediately *after* the Bicep deployment and *before* the script execution.

**1. Bicep Creation (Infrastructure):**
```bicep
// Link to Blob Storage 
resource splBlob 'Microsoft.Search/searchServices/sharedPrivateLinkResources@2024-03-01-preview' = {
  parent: searchService
  name: 'spl-blob'
  properties: {
    groupId: 'blob'
    privateLinkResourceId: storageAccount.id
    requestMessage: 'Auto-requested via Bicep'
  }
}

// Link to AI Services (Vision)
resource splVision 'Microsoft.Search/searchServices/sharedPrivateLinkResources@2024-03-01-preview' = {
  parent: searchService
  name: 'spl-vision'
  properties: {
    groupId: 'cognitiveservices'
    privateLinkResourceId: aiServicesAccount.id
    requestMessage: 'Auto-requested via Bicep'
  }
}

// Link to AI Services (OpenAI)
resource splOpenAI 'Microsoft.Search/searchServices/sharedPrivateLinkResources@2024-03-01-preview' = {
  parent: searchService
  name: 'spl-openai'
  properties: {
    groupId: 'openai'
    privateLinkResourceId: aiServicesAccount.id
    requestMessage: 'Auto-requested via Bicep'
  }
}
```

**2a. Azure DevOps YAML Approval Task:**
```yaml
- task: AzureCLI@2
  displayName: '✅ Approve Pending Shared Private Links'
  inputs:
    azureSubscription: 'Your-Service-Connection'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      echo "Approving pending connections on Storage Account..."
      STG_PENDING=$(az network private-endpoint-connection list --id $(StorageAccountId) --query "[?properties.privateLinkServiceConnectionState.status=='Pending'].id" -o tsv | tr -d '\r')
      for id in $STG_PENDING; do [ -n "$id" ] && az network private-endpoint-connection approve --id "$id" --description "Auto-approved"; done
      
      echo "Approving pending connections on AI Services Account..."
      AISVC_PENDING=$(az network private-endpoint-connection list --id $(AiServicesAccountId) --query "[?properties.privateLinkServiceConnectionState.status=='Pending'].id" -o tsv | tr -d '\r')
      for id in $AISVC_PENDING; do [ -n "$id" ] && az network private-endpoint-connection approve --id "$id" --description "Auto-approved"; done
```

**2b. GitHub Actions Approval Step:**
```yaml
- name: ✅ Approve Pending Shared Private Links
  uses: azure/cli@v2
  with:
    azcliversion: latest
    inlineScript: |
      echo "Approving pending connections on Storage Account..."
      STG_PENDING=$(az network private-endpoint-connection list --id ${{ env.STORAGE_ACCOUNT_ID }} --query "[?properties.privateLinkServiceConnectionState.status=='Pending'].id" -o tsv | tr -d '\r')
      for id in $STG_PENDING; do [ -n "$id" ] && az network private-endpoint-connection approve --id "$id" --description "Auto-approved"; done
      
      echo "Approving pending connections on AI Services Account..."
      AISVC_PENDING=$(az network private-endpoint-connection list --id ${{ env.AI_SERVICES_ACCOUNT_ID }} --query "[?properties.privateLinkServiceConnectionState.status=='Pending'].id" -o tsv | tr -d '\r')
      for id in $AISVC_PENDING; do [ -n "$id" ] && az network private-endpoint-connection approve --id "$id" --description "Auto-approved"; done
```

#### ⚠️ CRITICAL: Indexer Execution Environment
Even if Shared Private Links are Approved, Azure Search indexers default to an `Auto` execution environment, which may spin up a public multi-tenant worker. This public worker will ignore your private links and instantly fail with a `403 Forbidden` (Processing 0 items in ~200ms).

To force the indexer to tunnel through your VNet and use the Shared Private Links, the indexer JSON payload **must** include:
`"executionEnvironment": "private"` inside the `"configuration"` block.

*(Note: This configuration is already baked into `configure_azure_search_service_enterprise.sh`, but it causes a 2-5 minute "Cold Start" delay on the first run as Azure provisions the dedicated private node).*

---

### 🪲 Windows Agent Compatibility (ADO & GitHub Actions)
If this script is executed on a Windows-based runner (e.g., Azure DevOps Agent or GitHub Actions `windows-latest`), two common pipeline failures occur:

1. **Git Bash Path Mangling (`MissingSubscription` Error):** Git Bash converts Linux-style paths to Windows paths, corrupting Azure Resource IDs (e.g., `/subscriptions/...` becomes `C:/Program Files/Git/subscriptions/...`). 
   * **Fix:** The script internally sets `export MSYS_NO_PATHCONV=1` to prevent this. Do not remove this line.
2. **Carriage Returns (`\r`):** Git checkout on Windows injects `CRLF` line endings, shattering multi-line bash commands.
   * **Fix:** The pipeline task executing this script should use an `inlineScript` to sanitize the file with `sed -i 's/\r$//'` before running it.

---

### 🚀 Pipeline Execution Note (YAML Setup)
This script idempotently installs the Azure CLI `eventgrid` extension. Because CI/CD runners isolate the CLI environment, you must correctly configure your CLI task depending on your platform.

**Example 1: Azure DevOps Pipeline YAML:**
*(Requires `useGlobalConfig: true`)*
```yaml
- task: AzureCLI@2
  displayName: 'Deploy Search Pipeline Schema'
  inputs:
    azureSubscription: 'Your-Service-Connection'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      # 1. Strip Windows line endings (\r) to make it Linux-safe
      sed -i 's/\r$//' search-configuration/configure_azure_search_enterprise_pipeline.sh
      
      # 2. Execute
      bash search-configuration/configure_azure_search_enterprise_pipeline.sh <tenant-id> <sub-id> <rg> ...
    useGlobalConfig: true 
```

**Example 2: GitHub Actions Step:**
*(Uses the official `azure/cli@v2` action)*
```yaml
- name: Deploy Search Pipeline Schema
  uses: azure/cli@v2
  with:
    azcliversion: latest
    inlineScript: |
      # 1. Strip Windows line endings (\r) to make it Linux-safe
      sed -i 's/\r$//' search-configuration/configure_azure_search_enterprise_pipeline.sh
      
      # 2. Execute
      bash search-configuration/configure_azure_search_enterprise_pipeline.sh <tenant-id> <sub-id> <rg> ...
```

---

### 💻 Example Call
The script uses `az rest` with Entra ID tokens. No API keys are required as arguments.

```bash
./configure_azure_search_service_enterprise.sh \
  <tenant-id> \
  <subscription-id> \
  <resource-group> \
  <storage-account> \
  <container> \
  <ai-services-name> \
  <aoai-subdomain> \
  <aiservices-subdomain> \
  <search-name> \
  <virtual-dir> \
  <datasource-name> \
  <index-name> \
  <skillset-name> \
  <indexer-name> \
  <container-app-name> \
  <rbac-only-mode> \
  <enable-image-vectors>
```

---

### 🩺 Verification & Debugging Commands
If the indexer fails or you need to verify the Dual-Track architecture, run the following `az rest` commands from an authenticated terminal:

**1. Check Status & Throttling Warnings:**
```bash
# Check Status
az rest --method get --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexers/${SEARCH_INDEXER}/status?api-version=2024-11-01-preview" --query "lastResult.{Status:status, ItemsProcessed:itemsProcessed, ItemsFailed:itemsFailed}" --output table

# Check for hidden API throttling or truncated PDFs
az rest --method get --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexers/${SEARCH_INDEXER}/status?api-version=2024-11-01-preview" --query "lastResult.warnings" --output json
```

**2. Verify Parent Metadata (Image tags passed successfully):**
```bash
az rest --method post --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexes/${SEARCH_INDEX}/docs/search?api-version=2024-11-01-preview" --headers '{"Content-Type": "application/json"}' --body '{"search": "*", "filter": "doc_id ne null", "top": 1, "select": "doc_id, child_images, user_folder"}'
```

**3. Nuclear Reset (Clear transient failure history):**
```bash
az rest --method post --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexers/${SEARCH_INDEXER}/reset?api-version=2024-11-01-preview"
az rest --method post --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexers/${SEARCH_INDEXER}/run?api-version=2024-11-01-preview"
```
