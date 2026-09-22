# Azure AI Search Configuration Script

## File: `configure_azure_search_service_enterprise.sh`

### Responsibility Boundary
To maintain a clean separation of concerns, the deployment is split as follows:

1. **Infrastructure:** Creates the Search Service, Storage, and AI Services. Handles "outbound" Managed Identity roles.
2. **This Script:** Defines the Index schema, Skillsets, and Indexers. Handles "inbound" RBAC for the Container App and the Script Executor.

---

### Prerequisites
For this script to succeed, the infrastructure must ensure:

| Feature | Requirement |
| :--- | :--- |
| **Identity** | Search Service must have a `SystemAssigned` Managed Identity. |
| **Auth Mode** | The script supports both Hybrid (`aadOrApiKey`) and strict RBAC (`disableLocalAuth: true`). |
| **Outbound RBAC** | Search MI needs `Storage Blob Data Reader`, `Cognitive Services User`, and `Cognitive Services OpenAI User` on target resources. |
| **Model Sync** | `deploymentId` inside Skillset & Index JSONs must EXACTLY match the custom deployment name in Azure Foundry. `modelName` must remain the immutable base framework name (`text-embedding-3-large`). |
---

### Pre-Flight Environment Verification Checklist
Before running the configuration script in a new environment (especially PROD), verify the following Identity and Networking configurations directly in the Azure Portal.

#### 1. IAM & Role Assignments (The "Lock & Key" Check)
* **Search Service Identity Check:**
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

#### 3. AI Studio Deployment Synchronization (The Model Naming Check)
Before triggering the schema deployment, verify that your Azure OpenAI Studio deployment matches the definitions compiled by this script.
* **The Embedding Skill Alignment:**
  * Navigate to **Azure OpenAI Studio** -> **Deployments**.
  * Find the deployment running the `text-embedding-3-large` model.
  * **Requirement:** The exact string in the **Deployment name** column must match the `deploymentId` configuration property passed inside your Skillset and Index JSON profiles. 
  * **Critical Constraint:** Leave the `modelName` parameter inside your configuration schemas untouched as `text-embedding-3-large` (Azure Search relies on this static base model name to initialize tokenizer arrays, but relies on `deploymentId` for API routing).
* **The Backend App Suffix Warning:**
  * If this environment is also provisioning text/reasoning deployments (e.g., GPT-5.5) for the main application container:
  * **Requirement:** Ensure custom deployment names **do not end in a raw date string/timestamp** (e.g., `gpt-5.5-custom-name-2026-06-30` will trigger regex truncation failures in the app container; append a tracking label like `-v1` or `-prod` if a suffix is mandatory).

---
### 🔒 Secure Environments (Shared Private Links)
If the target environment has **Public Network Access disabled** on the Storage or AI resources, you must provision **Shared Private Link Resources** originating from the Search Service so it can securely access the data.

**Architectural Note (Unified vs. Split AI):**
* **Unified:** If you use a single Azure AI Multi-Service account for *both* embeddings and vision, you only need 2 Private Links (Storage + AI Account).
* **Split:** If you use a dedicated Azure OpenAI account for embeddings and a separate Multi-Service account for Vision/OCR, you need 3 Private Links.

**Required Shared Private Links (`groupId`):**
1. Blob Storage (`blob`) -> Target: Storage Account
2. AI Services / Vision (`cognitiveservices`) -> Target: Multi-Service AI Account (or the Unified Account)
3. AI Services / OpenAI (`openai` or `openai_account`) -> Target: Azure OpenAI Account (or the Unified Account)

#### Option A: Manual Workflow via Azure Portal (Current)
Deploying a Shared Private Link puts the connection into a `Pending` state. **These endpoints must be manually approved in the Azure Portal before this configuration script runs;** otherwise, skillset validation will fail with a `403 Forbidden`.

1. **Request Links:** In the Azure Portal, go to your Search Service -> **Networking** -> **Shared private access** and add the 2 or 3 links required for your architecture above.
2. **Approve Storage:** Go to your Storage Account -> **Networking** -> **Private endpoint connections** -> Check the pending request and click **Approve**.
3. **Approve AI Services:** Go to your AI Multi-Service Account (and Azure OpenAI account, if split) -> **Networking** -> **Private endpoint connections** -> Check the pending requests and click **Approve**.

Once all show as **Approved** in the Search Service, you may run this script.

#### Option B: Automated Workflow via Bicep & Pipeline (Future Reference)
To fully automate this in the future, add the links to the Bicep template and add an approval task to the CI/CD pipeline immediately *after* the Bicep deployment and *before* the script execution.

*(Note: If using a Unified architecture, simply pass the same Resource ID for both the OpenAI and Vision target properties; Bicep and the CLI loops will safely handle it).*

**1. Bicep Creation (Infrastructure):**
```bicep
// Link to Blob Storage 
resource splBlob 'Microsoft.Search/searchServices/sharedPrivateLinkResources@2024-03-01-preview' = {
  parent: searchService
  name: 'spl-blob'
  properties: {
    groupId: 'blob'
    privateLinkResourceId: storageAccountId
    requestMessage: 'Auto-requested via Bicep'
  }
}

// Link to AI Services (Vision)
resource splVision 'Microsoft.Search/searchServices/sharedPrivateLinkResources@2024-03-01-preview' = {
  parent: searchService
  name: 'spl-vision'
  properties: {
    groupId: 'cognitiveservices'
    privateLinkResourceId: aiVisionAccountId // Same as azureOpenAiAccountId if unified
    requestMessage: 'Auto-requested via Bicep'
  }
}

// Link to AI Services (OpenAI)
resource splOpenAI 'Microsoft.Search/searchServices/sharedPrivateLinkResources@2024-03-01-preview' = {
  parent: searchService
  name: 'spl-openai'
  properties: {
    groupId: 'openai'
    privateLinkResourceId: azureOpenAiAccountId // Same as aiVisionAccountId if unified
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
      
      echo "Approving pending connections on Azure OpenAI Account..."
      AOAI_PENDING=$(az network private-endpoint-connection list --id $(AzureOpenAiAccountId) --query "[?properties.privateLinkServiceConnectionState.status=='Pending'].id" -o tsv | tr -d '\r')
      for id in $AOAI_PENDING; do [ -n "$id" ] && az network private-endpoint-connection approve --id "$id" --description "Auto-approved"; done

      echo "Approving pending connections on AI Services (Vision) Account..."
      AISVC_PENDING=$(az network private-endpoint-connection list --id $(AiVisionAccountId) --query "[?properties.privateLinkServiceConnectionState.status=='Pending'].id" -o tsv | tr -d '\r')
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
      
      echo "Approving pending connections on Azure OpenAI Account..."
      AOAI_PENDING=$(az network private-endpoint-connection list --id ${{ env.AZURE_OPENAI_ACCOUNT_ID }} --query "[?properties.privateLinkServiceConnectionState.status=='Pending'].id" -o tsv | tr -d '\r')
      for id in $AOAI_PENDING; do [ -n "$id" ] && az network private-endpoint-connection approve --id "$id" --description "Auto-approved"; done

      echo "Approving pending connections on AI Services (Vision) Account..."
      AISVC_PENDING=$(az network private-endpoint-connection list --id ${{ env.AI_VISION_ACCOUNT_ID }} --query "[?properties.privateLinkServiceConnectionState.status=='Pending'].id" -o tsv | tr -d '\r')
      for id in $AISVC_PENDING; do [ -n "$id" ] && az network private-endpoint-connection approve --id "$id" --description "Auto-approved"; done
```

#### ⚠️ CRITICAL: Indexer Execution Environment
Even if Shared Private Links are Approved, Azure Search indexers default to an `Auto` execution environment, which may spin up a public multi-tenant worker. This public worker will ignore your private links and instantly fail with a `403 Forbidden` (Processing 0 items in ~200ms).

To force the indexer to tunnel through your VNet and use the Shared Private Links, the indexer JSON payload **must** include:
`"executionEnvironment": "private"` inside the `"configuration"` block.

Note: This configuration is already baked into `configure_azure_search_service_enterprise.sh`, but it causes a 2-5 minute "Cold Start" delay on the first run as Azure provisions the dedicated private node.

---

### Windows Agent Compatibility (ADO & GitHub Actions)
If this script is executed on a Windows-based runner (e.g., Azure DevOps Agent or GitHub Actions `windows-latest`), two common pipeline failures occur:

1. **Git Bash Path Mangling (`MissingSubscription` Error):** Git Bash converts Linux-style paths to Windows paths, corrupting Azure Resource IDs (e.g., `/subscriptions/...` becomes `C:/Program Files/Git/subscriptions/...`). 
   * **Fix:** The script internally sets `export MSYS_NO_PATHCONV=1` to prevent this. Do not remove this line.
2. **Carriage Returns (`\r`):** Git checkout on Windows injects `CRLF` line endings, shattering multi-line bash commands.
   * **Fix:** The pipeline task executing this script should use an `inlineScript` to sanitize the file with `sed -i 's/\r$//'` before running it.

---

### Pipeline Execution Note (YAML Setup)
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

### Example Call
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

### Verification & Debugging Commands
If the indexer fails or you need to verify the Dual-Track architecture and data completeness, run the following `az rest` commands from an authenticated terminal.

The commands explicitly use `--resource https://search.azure.com` so Azure CLI requests the correct Azure AI Search access token for the custom Search service endpoint.

**1. Check Status & Throttling Warnings:**
```bash
# Check the latest indexer execution status and progress
az rest --method get \
  --resource https://search.azure.com \
  --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexers/${SEARCH_INDEXER}/status?api-version=2024-11-01-preview" \
  --query "lastResult.{Status:status,Start:startTime,End:endTime,ItemsProcessed:itemsProcessed,ItemsFailed:itemsFailed}" \
  --output table

# Check detailed warnings and errors.
# This can surface oversized/truncated documents, enrichment/projection issues,
# throttling, unsupported content, and other per-document/indexer problems.
az rest --method get \
  --resource https://search.azure.com \
  --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexers/${SEARCH_INDEXER}/status?api-version=2024-11-01-preview" \
  --query "lastResult.{Errors:errors,Warnings:warnings}" \
  --output json
```

**2. Verify Soft Delete Configuration:**
Ensure the datasource is actively listening for soft-deleted blobs so ghost data doesn't persist in the search index.
```bash
az rest --method get \
  --resource https://search.azure.com \
  --url "https://$SEARCH_SERVICE_NAME.search.windows.net/datasources/${SEARCH_DATASOURCE}?api-version=2024-11-01-preview" \
  --query "dataDeletionDetectionPolicy" \
  --output json
```

**3. Verify Data Completeness (Chunk Counts):**
Shows how many projected text chunks were generated per filename.

Only chunk rows are counted (`parent_id ne null`), so parent-document rows do not inflate the counts. If a large document yields unexpectedly few chunks, inspect the indexer warnings and source document; this may indicate extraction, truncation, or enrichment problems.

Note: if different databases contain files with the exact same `file_name`, the first filename-only query aggregates those files together. Use the user-specific query or the SharePoint folder verification command below when investigating a specific scope.

```bash
# Get a list of unique filenames and their chunk counts
az rest --method post \
  --resource https://search.azure.com \
  --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexes/${SEARCH_INDEX}/docs/search?api-version=2024-11-01-preview" \
  --headers '{"Content-Type": "application/json"}' \
  --body '{
    "search": "*",
    "filter": "parent_id ne null",
    "facets": ["file_name,count:5000"],
    "top": 0
  }' | jq '.["@search.facets"].file_name'

# Optional: Filter the chunk count by a specific user / organization.
# Examples:
#   TARGET_USER="GLOBAL"
#   TARGET_USER="ORG_<company>"
#   TARGET_USER="<personal_user_id>"
TARGET_USER="GLOBAL"

az rest --method post \
  --resource https://search.azure.com \
  --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexes/${SEARCH_INDEX}/docs/search?api-version=2024-11-01-preview" \
  --headers '{"Content-Type": "application/json"}' \
  --body "{
    \"search\": \"*\",
    \"filter\": \"user_id eq '${TARGET_USER}' and parent_id ne null\",
    \"facets\": [\"file_name,count:500\"],
    \"top\": 0
  }" | jq '.["@search.facets"].file_name'
```

**4. Verify Parent Metadata (Relational Logic):**
Confirms that parent document rows were created correctly.

`child_images` may legitimately be empty for documents that have no retained extracted images, so an empty `child_images` value by itself is not an indexing failure.
```bash
az rest --method post \
  --resource https://search.azure.com \
  --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexes/${SEARCH_INDEX}/docs/search?api-version=2024-11-01-preview" \
  --headers '{"Content-Type": "application/json"}' \
  --body '{
    "search": "*",
    "filter": "doc_id ne null",
    "top": 5,
    "select": "doc_id,file_name,child_images,user_folder"
  }'
```

**5. Verify Document Chunking (Parallel Track):**
Proves the text was successfully split into smaller text chunks and linked back to the parent document.
```bash
az rest --method post \
  --resource https://search.azure.com \
  --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexes/${SEARCH_INDEX}/docs/search?api-version=2024-11-01-preview" \
  --headers '{"Content-Type": "application/json"}' \
  --body '{
    "search": "*",
    "filter": "parent_id ne null",
    "top": 1,
    "select": "chunk_id,parent_id,chunk"
  }'
```

**6. Verify SharePoint-Mirrored Database Completeness (SPAuto):**
For deployments with SharePoint mirroring enabled, verifies how many SharePoint-mirrored parent documents and distinct `SPAuto_*` database folders are currently represented in Azure AI Search.

Run this after the SharePoint mirror has finished and the triggered Search indexer run has completed.

Compare the resulting folder/document counts with `_sync_state/sharepoint_delta_state.json` and the corresponding `custom_databases/ORG_<company>/SPAuto_*` folders in Blob Storage.

The `SPAuto_` prefix check is intentionally performed locally with `jq`. Do not use `startswith()` inside the Azure AI Search `$filter`, because that function is not supported by the Search OData filter syntax.

```bash
TARGET_USER="ORG_<company>"

az rest --method post \
  --resource https://search.azure.com \
  --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexes/${SEARCH_INDEX}/docs/search?api-version=2024-11-01-preview" \
  --headers '{"Content-Type": "application/json"}' \
  --body "{
    \"search\": \"*\",
    \"filter\": \"user_id eq '${TARGET_USER}' and doc_id ne null\",
    \"facets\": [\"user_folder,count:5000\"],
    \"top\": 0
  }" \
| jq '
  [(.["@search.facets"].user_folder // [])[]
   | select(.value | startswith("SPAuto_"))] as $sp
  | {
      SPAutoFolders: ($sp | length),
      SPAutoParentDocuments: ($sp | map(.count) | add // 0),
      Folders: $sp
    }
'
```

**7. Nuclear Reset / Full Reprocessing:**
Force a full re-processing if you change AI enrichment logic, chunking strategy, or otherwise need to completely reprocess the datasource after an upstream failure.

`reset` clears the indexer's internal change-tracking/high-water mark. The subsequent `run` command performs the actual reprocessing.

This is not required for normal incremental indexing and should not be used as a routine diagnostic command.

Note: resetting and rerunning the indexer does not by itself remove arbitrary orphaned Search documents that no longer have a corresponding source document. Normal Blob deletions should instead be handled through the configured soft-delete detection policy.

```bash
az rest --method post \
  --resource https://search.azure.com \
  --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexers/${SEARCH_INDEXER}/reset?api-version=2024-11-01-preview"

az rest --method post \
  --resource https://search.azure.com \
  --url "https://$SEARCH_SERVICE_NAME.search.windows.net/indexers/${SEARCH_INDEXER}/run?api-version=2024-11-01-preview"
```
