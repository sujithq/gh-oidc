# gh-oidc

GitHub OIDC Sample for App Registration and Managed Identity.
For each there is a script for creating the Federated Credential and a basic pipeline using the Federated Credential.
Depending on the GitHub Entity Type you need to provide additional info (environment, branch of tag) except for pull_request

Run the script and use the output to set the GitHub Action Secrets

* secrets.AZURE_CLIENT_ID or secrets.AZURE_CLIENT_ID_MI
* secrets.AZURE_TENANT_ID
* secrets.AZURE_SUBSCRIPTION_ID

Add the corresponding pipelines.

## App Registration

```bash
# Variables
AZURE_SUBSCRIPTION_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"           # Azure Subscription Id

# Create App Registration
result=$(az ad sp create-for-rbac --role="Contributor" --scopes="/subscriptions/$AZURE_SUBSCRIPTION_ID")

# Get AAD Application Id
AAD_CLIENT_ID=$(echo $result | jq -r '.appId')
AAD_TENANT_ID=$(echo $result | jq -r '.tenant')

GH_ORG="sujithq"              # GitHub Organization
GH_REPO="gh-oidc"             # GitHub Repository
GH_ENTITY_TYPE="branch"  # GitHub Entity Type one of [environment|branch|pull_request|tag]
GH_ENVIRONMENT="production"   # GitHub Environment
GH_BRANCH="main"              # GitHub Branch
GH_TAG="v1.0.0"               # GitHub Tag

PARAMS_SUBJECT="repo:${GH_ORG}/${GH_REPO}:"

case $GH_ENTITY_TYPE in
    "environment")
        PARAMS_SUBJECT+="environment:${GH_ENVIRONMENT}"
        ;;
    "branch")
        PARAMS_SUBJECT+="ref:refs/heads/${GH_BRANCH}"
        ;;
    "pull_request")
        PARAMS_SUBJECT+="pull_request"
        ;;
    "tag")
        PARAMS_SUBJECT+="ref:refs/tags/${GH_TAG}"
        ;;
    *)
        echo "Invalid GitHub Entity Type"
        exit 1
        ;;
esac


PARAMS_NAME="$GH_ORG-$GH_REPO-$GH_ENTITY_TYPE-federated-identity"
PARAMS_ISSUER="https://token.actions.githubusercontent.com"
PARAMS_DESCRIPTION="Federation for GitHub $GH_ORG|$GH_REPO|$GH_ENTITY_TYPE"

# Compose a params.json configuration file
cat <<EOF > params.json
{
  "name": "${PARAMS_NAME}",
  "issuer": "${PARAMS_ISSUER}",
  "subject": "${PARAMS_SUBJECT}",
  "description": "${PARAMS_DESCRIPTION}",
  "audiences": [
    "api://AzureADTokenExchange"
  ]
}
EOF

# Create Federated Credential
result=$(az ad app federated-credential create --id $AAD_CLIENT_ID --parameters params.json)

echo "Successfully created"
echo "Secrets.AZURE_CLIENT_ID to $AAD_CLIENT_ID"
echo "Set secrets.AZURE_TENANT_ID to $AAD_TENANT_ID"
echo "Set secrets.AZURE_SUBSCRIPTION_ID to $AZURE_SUBSCRIPTION_ID"

```
```yml
name: WIF App registration with federation

permissions:
  id-token: write # This is required for requesting the JWT
  contents: read  # This is required for actions/checkout

on:
  push:
    branches:
    - main
  workflow_dispatch:
jobs:
    job01:
        runs-on: ubuntu-latest
        steps:
        - uses: Azure/login@v1
          with:
            client-id: ${{ secrets.AZURE_CLIENT_ID }}
            tenant-id: ${{ secrets.AZURE_TENANT_ID }}
            subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
        - name: Get Subscription Name
          uses: azure/CLI@v1
          with:
            inlineScript: |
                # Show current account information
                info=$(az account show)
                # Get Subscription Name
                SUBSCRIPTION_NAME=$(echo $info | jq -r '.name')
                echo "Subscription name is $SUBSCRIPTION_NAME"
```

## Managed Identity

```bash
# Variables
AZURE_SUBSCRIPTION_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"  # Azure Subscription Id
AZURE_RG="rg-gh-oidc"                                         # Azure Resource Group
AZURE_LOCATION="westeurope"                                   # Azure location
ID_NAME="id-gh-oidc"                                          # Managed Identity Name

# Crceate Resource Group
az group create --resource-group $AZURE_RG --location $AZURE_LOCATION

# Create Managed Identity
result=$(az identity create --name $ID_NAME --resource-group $AZURE_RG)

MI_ID=$(echo $result | jq -r '.id')                       # Get Managed Identity Id
AAD_CLIENT_ID=$(echo $result | jq -r '.clientId')         # Get AAD Application Id
AAD_PRINICIPAL_ID=$(echo $result | jq -r '.principalId')  # Get AAD Application Id
AAD_TENANT_ID=$(echo $result | jq -r '.tenantId')         # Get AAD Application Id

# Role Assignment
#
# With Graph Permissions (uncomment below)
# az role assignment create --role "Contributor" --assignee $MI_ID --scope /subscriptions/$AZURE_SUBSCRIPTION_ID
# 
# Without Graph Permissions (uncomment below)
az role assignment create --role "Contributor" --assignee-object-id $AAD_PRINICIPAL_ID --assignee-principal-type ServicePrincipal --scope /subscriptions/$AZURE_SUBSCRIPTION_ID

GH_ORG="sujithq"              # GitHub Organization
GH_REPO="gh-oidc"             # GitHub Repository
GH_ENTITY_TYPE="branch"       # GitHub Entity Type one of [environment|branch|pull_request|tag]
GH_ENVIRONMENT="production"   # GitHub Environment
GH_BRANCH="main"              # GitHub Branch
GH_TAG="v1.0.0"               # GitHub Tag

PARAMS_SUBJECT="repo:${GH_ORG}/${GH_REPO}:"

case $GH_ENTITY_TYPE in
    "environment")
        PARAMS_SUBJECT+="environment:${GH_ENVIRONMENT}"
        ;;
    "branch")
        PARAMS_SUBJECT+="ref:refs/heads/${GH_BRANCH}"
        ;;
    "pull_request")
        PARAMS_SUBJECT+="pull_request"
        ;;
    "tag")
        PARAMS_SUBJECT+="ref:refs/tags/${GH_TAG}"
        ;;
    *)
        echo "Invalid GitHub Entity Type"
        exit 1
        ;;
esac


PARAMS_NAME="$GH_ORG-$GH_REPO-$GH_ENTITY_TYPE-federated-identity"
PARAMS_ISSUER="https://token.actions.githubusercontent.com"
PARAMS_DESCRIPTION="Federation for GitHub $GH_ORG|$GH_REPO|$GH_ENTITY_TYPE"

# Create Federated Credential
result=$(az identity federated-credential create --name $PARAMS_NAME --identity-name $ID_NAME --resource-group $AZURE_RG --issuer $PARAMS_ISSUER --subject $PARAMS_SUBJECT --audiences "api://AzureADTokenExchange")

echo "Successfully created"
echo "Set secrets.AZURE_CLIENT_ID_MI to $AAD_CLIENT_ID"
echo "Set secrets.AZURE_TENANT_ID to $AAD_TENANT_ID"
echo "Set secrets.AZURE_SUBSCRIPTION_ID to $AZURE_SUBSCRIPTION_ID"
```

```yml
name: WIF Managed Identity with federation

permissions:
  id-token: write # This is required for requesting the JWT
  contents: read  # This is required for actions/checkout

on:
  push:
    branches:
    - main
  workflow_dispatch:
jobs:
    job01:
        runs-on: ubuntu-latest
        steps:
        - uses: Azure/login@v1
          with:
            client-id: ${{ secrets.AZURE_CLIENT_ID_MI }}
            tenant-id: ${{ secrets.AZURE_TENANT_ID }}
            subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
        - name: Get Subscription Name
          uses: azure/CLI@v1
          with:
            inlineScript: |
                # Show current account information
                info=$(az account show)
                # Get Subscription Name
                SUBSCRIPTION_NAME=$(echo $info | jq -r '.name')
                echo "Subscription name is $SUBSCRIPTION_NAME"
```
