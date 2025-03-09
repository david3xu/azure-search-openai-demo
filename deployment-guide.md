# Azure OpenAI RAG Demo Deployment Guide

This document details the steps taken to deploy the Azure OpenAI RAG (Retrieval Augmented Generation) demo application using existing Azure resources.

## Prerequisites

- A dev container environment (VS Code Remote Container, GitHub Codespaces, etc.)
- Existing Azure resources for the application
- Azure CLI and Azure Developer CLI (azd) installed in the container

## 1. Authentication

First, authenticate with both Azure CLI and Azure Developer CLI:

```bash
# Authenticate with Azure Developer CLI
azd auth login --use-device-code

# Authenticate with Azure CLI
az login --use-device-code

# Select the appropriate subscription
# In our case, we selected: Microsoft Azure Sponsorship (ccc6af52-5928-4dbe-8ceb-fa794974a30f)
```

## 2. Resource Discovery

Identify the existing Azure resources that will be used for deployment:

```bash
# List resource groups
az group list --query "[].{Name:name, Location:location}" -o table

# Output:
# Name                      Location
# ------------------------  ----------
# DefaultResourceGroup-EUS  eastus
# rg-ragagentic             eastus
```

Examine resources in the target resource group:

```bash
# List resources in the resource group
az resource list --resource-group rg-ragagentic --query "[].{Name:name, Type:type}" -o table

# Output:
# Name                                  Type
# ------------------------------------  ------------------------------------------------
# rag-demo-aca-identity                 Microsoft.ManagedIdentity/userAssignedIdentities
# ragagenticproject-openai              Microsoft.CognitiveServices/accounts
# cog-di-uiuvvh33kzw3g                  Microsoft.CognitiveServices/accounts
# ragagenticprojectstorage              Microsoft.Storage/storageAccounts
# log-uiuvvh33kzw3g                     Microsoft.OperationalInsights/workspaces
# appi-uiuvvh33kzw3g                    Microsoft.Insights/components
# ragagenticproject-search              Microsoft.Search/searchServices
# dash-uiuvvh33kzw3g                    Microsoft.Portal/dashboards
# ragdemoacruiuvvh33kzw3g               Microsoft.ContainerRegistry/registries
# rag-demo-aca-env                      Microsoft.App/managedEnvironments
# capps-backend-uiuvvh33kzw3g           Microsoft.App/containerApps
# Application Insights Smart Detection  microsoft.insights/actiongroups
```

## 3. Environment Setup

Create a new Azure Developer CLI environment and configure it to use existing resources:

```bash
# Create a new environment
azd env new existing-resources

# Configure the environment with existing resource information
azd env set AZURE_RESOURCE_GROUP rg-ragagentic
azd env set AZURE_LOCATION eastus

# OpenAI Service
azd env set AZURE_OPENAI_SERVICE ragagenticproject-openai
azd env set AZURE_OPENAI_RESOURCE_GROUP rg-ragagentic
azd env set AZURE_OPENAI_LOCATION eastus

# Azure AI Search
azd env set AZURE_SEARCH_SERVICE ragagenticproject-search
azd env set AZURE_SEARCH_SERVICE_RESOURCE_GROUP rg-ragagentic

# Storage Account
azd env set AZURE_STORAGE_ACCOUNT ragagenticprojectstorage

# Document Intelligence
azd env set AZURE_DOCUMENTINTELLIGENCE_SERVICE cog-di-uiuvvh33kzw3g

# Deployment Target (Azure Container Apps)
azd env set DEPLOYMENT_TARGET containerapps

# Container Registry
azd env set AZURE_CONTAINER_REGISTRY ragdemoacruiuvvh33kzw3g
```

## 4. Deployment

Initiate the deployment using the `azd up` command:

```bash
azd up
```

This command:
1. Validates the environment configuration
2. Prompts for any missing parameters
3. Executes pre-build hooks (building frontend assets)
4. Packages the application into container images
5. Pushes the container images to the Azure Container Registry
6. Deploys the application to the Azure Container App

During the execution, you will be prompted to:
- Select the Azure subscription to use
- Confirm the location for resources

### Deployment Results

In our deployment, the following steps completed successfully:

1. Resource group validation
2. Azure OpenAI service configuration and model deployments
3. Storage account setup
4. Search service configuration
5. Log Analytics workspace creation
6. Document Intelligence setup
7. Container Registry configuration
8. Container App environment preparation
9. Data indexing (all sample documents were processed)
   - Documents were split into sections
   - Embeddings were created for all content
   - Content was uploaded to the search index

### Deployment Error Encountered

We encountered the following error during the final deployment stage:

```
ERROR: error executing step command 'deploy --all': getting target resource: 
expecting only '1' resource tagged with 'azd-service-name: backend', but found '2'. 
Ensure a unique service resource is correctly tagged in your infrastructure 
configuration, and rerun provision
```

This error occurs because our existing environment already had a container app tagged with 'azd-service-name: backend', and the deployment tried to create another one with the same tag.

## 5. Troubleshooting the Deployment Error

To resolve the tag conflict error, we used the approach of removing the existing tag from the current container app:

### Checking the container app resource ID

```bash
az containerapp show --name capps-backend-uiuvvh33kzw3g --resource-group rg-ragagentic --query id -o tsv

# Output:
# /subscriptions/ccc6af52-5928-4dbe-8ceb-fa794974a30f/resourceGroups/rg-ragagentic/providers/Microsoft.App/containerapps/capps-backend-uiuvvh33kzw3g
```

### Checking the current tags

```bash
az resource show --ids /subscriptions/ccc6af52-5928-4dbe-8ceb-fa794974a30f/resourceGroups/rg-ragagentic/providers/Microsoft.App/containerapps/capps-backend-uiuvvh33kzw3g --query tags

# Output:
# {
#   "azd-env-name": "rag-demo",
#   "azd-service-name": "backend"
# }
```

### Removing the azd-service-name tag

```bash
az resource update --ids /subscriptions/ccc6af52-5928-4dbe-8ceb-fa794974a30f/resourceGroups/rg-ragagentic/providers/Microsoft.App/containerapps/capps-backend-uiuvvh33kzw3g --set tags."azd-service-name"=""
```

### Verifying the tag was removed

```bash
az resource show --ids /subscriptions/ccc6af52-5928-4dbe-8ceb-fa794974a30f/resourceGroups/rg-ragagentic/providers/Microsoft.App/containerapps/capps-backend-uiuvvh33kzw3g --query tags

# Output:
# {
#   "azd-env-name": "rag-demo",
#   "azd-service-name": ""
# }
```

### Running the deployment again

After removing the tag, we ran the deployment again:

```bash
azd deploy
```

The deployment completed successfully this time.

## 6. Post-Deployment

After successful deployment:

- The application is now accessible at: 
  ```
  https://capps-backend-jxmgxni44tt5o.purplesea-143e83f7.eastus.azurecontainerapps.io
  ```

- The app is connected to all the configured Azure services:
  - OpenAI service for language model capabilities
  - Azure AI Search for document indexing and retrieval
  - Storage Account for document storage
  - Document Intelligence for processing documents

### OpenAI Models Used

The deployment uses the following Azure OpenAI models:

1. **Chat Completion Model**:
   - Service: `ragagenticproject-openai`
   - Deployment Name: `gpt-35-turbo-16k` (configured as `AZURE_OPENAI_CHATGPT_DEPLOYMENT`)
   - Model: GPT-3.5 Turbo with 16K context window
   - Used for: Answering user queries and generating responses based on retrieved content

2. **Embedding Model**:
   - Service: `ragagenticproject-openai`
   - Deployment Name: `text-embedding` (configured as `AZURE_OPENAI_EMB_DEPLOYMENT`)
   - Model: `text-embedding-ada-002`
   - Dimensions: 1536
   - Used for: Converting text into vector embeddings for semantic search

You can verify the model configurations in the Azure portal or by checking the container app's environment variables:

```bash
# Check the models configured in the container app's environment variables
az containerapp show --name capps-backend-jxmgxni44tt5o --resource-group rg-ragagentic --query "properties.template.containers[0].env[?name=='AZURE_OPENAI_CHATGPT_DEPLOYMENT' || name=='AZURE_OPENAI_EMB_DEPLOYMENT' || name=='AZURE_OPENAI_CHATGPT_MODEL']" -o table
```

To use different models or deployments, you can update the environment variables:

```bash
# Update the chat model deployment
azd env set AZURE_OPENAI_CHATGPT_DEPLOYMENT your-gpt4-deployment-name
azd env set AZURE_OPENAI_CHATGPT_MODEL gpt-4

# Apply the changes
azd deploy
```

## Azure Resources Used

| Resource Type | Resource Name | Purpose |
|---------------|---------------|---------|
| Resource Group | rg-ragagentic | Container for all resources |
| OpenAI Service | ragagenticproject-openai | Provides large language model capabilities |
| Azure AI Search | ragagenticproject-search | Indexes and retrieves document content |
| Storage Account | ragagenticprojectstorage | Stores uploaded documents |
| Document Intelligence | cog-di-uiuvvh33kzw3g | Processes and extracts text from documents |
| Container Registry | ragdemoacruiuvvh33kzw3g | Stores container images |
| Container App Environment | rag-demo-aca-env | Hosts the containerized application |
| Container App | capps-backend-jxmgxni44tt5o | The deployed application itself |

## Model Upgrades

### Upgrading to o3-mini Model

The o3-mini model offers enhanced reasoning capabilities, function calling, and tools support while being cost-efficient. Here's how to upgrade your existing deployment if the model is available in your region:

> **Note**: Model availability varies by region. Check which models are available in your OpenAI service region before attempting to deploy a new model.

1. **Check Model Availability**:
   ```bash
   # List available models in your Azure OpenAI service
   az cognitiveservices account deployment list --name ragagenticproject-openai --resource-group rg-ragagentic -o table
   ```

   If the desired model is not listed, you may need to:
   - Wait for the model to become available in your region
   - Create a new Azure OpenAI service in a region where the model is available
   - Use an alternative model that is available

2. **Create o3-mini Deployment in Azure OpenAI Service** (if available):
   - Go to Azure Portal → Azure OpenAI service (`ragagenticproject-openai`)
   - Select "Model deployments" → "+ Create new deployment"
   - Choose "o3-mini" from the model list
   - Name the deployment `o3-mini`
   - Set appropriate capacity units based on your needs
   - Click "Create"

3. **Update Environment Variables**:
   ```bash
   # Select the existing environment
   azd env select existing-resources
   
   # Set new model configuration
   azd env set AZURE_OPENAI_CHATGPT_MODEL o3-mini
   azd env set AZURE_OPENAI_CHATGPT_DEPLOYMENT o3-mini
   azd env set AZURE_OPENAI_REASONING_EFFORT high
   
   # Verify settings
   azd env get-values
   ```

4. **Deploy the Changes**:
   ```bash
   azd deploy
   ```

5. **If Deployment Doesn't Apply Changes, Update Container App Directly**:
   ```bash
   # Update the container app directly
   az containerapp update --name capps-backend-jxmgxni44tt5o --resource-group rg-ragagentic --set-env-vars AZURE_OPENAI_CHATGPT_MODEL=o3-mini AZURE_OPENAI_CHATGPT_DEPLOYMENT=o3-mini AZURE_OPENAI_REASONING_EFFORT=high
   ```

6. **Verify the Update**:
   ```bash
   # Verify the environment variables were updated
   az containerapp show --name capps-backend-jxmgxni44tt5o --resource-group rg-ragagentic --query "properties.template.containers[0].env[?name=='AZURE_OPENAI_CHATGPT_DEPLOYMENT' || name=='AZURE_OPENAI_CHATGPT_MODEL']" -o table
   ```

**Benefits of o3-mini**:
- Enhanced reasoning and problem-solving capabilities
- Support for function calling and developer messages
- Large context window (200K tokens)
- Better performance on complex tasks while being cost-efficient
- Advanced coding capabilities and instruction following

### Using Alternative Models

If o3-mini isn't available in your region, you can use alternative models:

```bash
# Example: Updating to use gpt-35-turbo-16k (if available)
az containerapp update --name capps-backend-jxmgxni44tt5o --resource-group rg-ragagentic --set-env-vars AZURE_OPENAI_CHATGPT_MODEL=gpt-35-turbo-16k AZURE_OPENAI_CHATGPT_DEPLOYMENT=gpt-35-turbo-16k
```

## Container App Management

### Checking Active Container Apps

You can list all container apps in your resource group:

```bash
# List all container apps with their status and URLs
az containerapp list --resource-group rg-ragagentic --query "[].{name:name, status:properties.runningStatus, url:properties.configuration.ingress.fqdn}" -o table
```

### Deleting Unused Container Apps

If you have multiple container apps and want to clean up unused ones:

```bash
# Delete a specific container app
az containerapp delete --name CONTAINER_APP_NAME --resource-group rg-ragagentic
```

## Data Management

### Uploading New Data

After your application is deployed, you may want to add new documents to your search index. The application includes built-in tools to process and index new documents using the same pipelines that were used during deployment.

#### Supported Document Formats

You can upload and process various document formats:
- PDF
- DOCX
- PPTX
- HTML
- TXT
- JSON
- Markdown (MD)

#### Steps to Upload New Data

1. **Add your new data files to the data directory:**
   
   ```bash
   # Create a new folder for your data (optional)
   mkdir -p data/new_documents
   
   # Copy your files into the data directory
   # For example:
   # cp /path/to/your/files/*.pdf data/new_documents/
   ```

2. **Make sure you're using the correct environment:**

   ```bash
   # Select the environment with your Azure resources
   azd env select existing-resources
   ```

3. **Process all data (default behavior):**

   ```bash
   # Process all documents in the data directory and subdirectories
   ./scripts/prepdocs.sh --category your-category-name
   ```

   This script will:
   - Extract text from documents using Azure Document Intelligence
   - Split documents into sections
   - Generate embeddings using Azure OpenAI
   - Upload documents to Azure Storage
   - Index the content in Azure AI Search

4. **Process only specific subdirectories:**

   The prepdocs.sh script has a hardcoded file pattern `'./data/*'` that processes all files in the data directory. To process only specific subdirectories, you need to temporarily modify the script:

   ```bash
   # 1. Create a backup of the original script
   cp ./scripts/prepdocs.sh ./scripts/prepdocs.sh.bak

   # 2. Modify the script to target only the specific subdirectory
   sed -i 's|./data/\*|./data/new_documents/specific-folder/\*|' ./scripts/prepdocs.sh

   # 3. Run the script with your desired options
   ./scripts/prepdocs.sh --category your-category-name

   # 4. Restore the original script when done
   mv ./scripts/prepdocs.sh.bak ./scripts/prepdocs.sh
   ```

   Replace `specific-folder` with the name of your subdirectory containing the files you want to process.

### Data Files and Version Control

The repository is configured to exclude data files from version control. This ensures that:

1. Large document files don't bloat the repository size
2. Sensitive or proprietary documents aren't accidentally shared
3. Repository operations remain fast and efficient

The `.gitignore` file includes rules to exclude:
- The entire `data/new_documents/` directory
- Common document formats in any data subdirectory
- Backup files (*.bak)

If you need to share specific data files with collaborators, consider using:
- A shared cloud storage location
- Azure Storage with controlled access
- Separate data repositories for large datasets

### Advanced Options for Data Processing

The prepdocs script supports several options to customize processing:

```bash
# Use a specific category for documents (useful for filtering in the frontend)
./scripts/prepdocs.sh --category marketing

# Skip uploading individual file blobs (saves storage)
./scripts/prepdocs.sh --skipblobs

# Use a custom search index (specify index in environment variables first)
azd env set AZURE_SEARCH_INDEX my-custom-index
./scripts/prepdocs.sh

# Remove all existing documents before indexing
./scripts/prepdocs.sh --removeall

# Combine multiple options
./scripts/prepdocs.sh --category finance --skipblobs
```

## Troubleshooting

If you encounter issues during deployment:

1. Check the azd logs for specific error messages
2. Verify that all required environment variables are set correctly
3. Ensure that the existing resources have the necessary permissions
4. Check if the container registry allows pushing from your current environment

## Useful Commands

```bash
# View environment variables
azd env get-values

# Check deployment status
az containerapp show --name capps-backend-jxmgxni44tt5o --resource-group rg-ragagentic

# View deployment logs
az containerapp logs show --name capps-backend-jxmgxni44tt5o --resource-group rg-ragagentic

# Update a specific environment variable
azd env set VARIABLE_NAME new_value

# Redeploy after code changes without reprovisioning resources
azd deploy
``` 