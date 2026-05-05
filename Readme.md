# Create a Foundry with BYON

```
#Create env variables
SUB="68b49ef1-514f-4d17-9849-26b3e9e1a838"
RG="my-foundry-rg"
LOCATION="swedencentral"
VNET="my-foundry-vnet"
ACCOUNT="my-foundry-ak"
PROJECT="my-foundry-project"
STORAGE="myfoundrystorageak"
COSMOS="my-foundry-cosmosak"
SEARCH="my-foundry-searchak"
KV="my-foundry-kvak"

# Step 1 — Resource Group
az group create --name $RG --location $LOCATION

# Step 2 — VNet & Subnets
az network vnet create \
  --name $VNET --resource-group $RG \
  --location $LOCATION --address-prefix 10.0.0.0/16

# Azure Firewall subnet (name is mandatory)
az network vnet subnet create \
  --name AzureFirewallSubnet --vnet-name $VNET \
  --resource-group $RG --address-prefix 10.0.0.0/26

# Foundry private endpoint subnet
az network vnet subnet create \
  --name snet-foundry-pe --vnet-name $VNET \
  --resource-group $RG --address-prefix 10.0.1.0/24 \
  --disable-private-endpoint-network-policies true

# Agent subnet — /24, delegated (use snet-agent-2, snet-agent is tainted)
az network vnet subnet create \
  --name snet-agent-2 --vnet-name $VNET \
  --resource-group $RG --address-prefix 10.0.4.0/24 \
  --delegations Microsoft.App/environments

# Connected services PE subnet
az network vnet subnet create \
  --name snet-services-pe --vnet-name $VNET \
  --resource-group $RG --address-prefix 10.0.3.0/24 \
  --disable-private-endpoint-network-policies true


# Step 3 — Azure Firewall + UDR
az network public-ip create \
  --name pip-firewall --resource-group $RG \
  --location $LOCATION --sku Standard

az network firewall create \
  --name my-firewall --resource-group $RG \
  --location $LOCATION --sku AZFW_VNet --tier Standard

az network firewall ip-config create \
  --name fw-ipconfig --firewall-name my-firewall \
  --resource-group $RG --public-ip-address pip-firewall \
  --vnet-name $VNET

FW_IP=$(az network firewall show \
  --name my-firewall --resource-group $RG \
  --query "ipConfigurations[0].privateIPAddress" -o tsv)

echo "Firewall IP: $FW_IP"

FW_IP=$(echo "$FW_IP" | tr -d '\r')

az network route-table create \
  --name rt-foundry --resource-group $RG --location $LOCATION

az network route-table route create \
  --route-table-name rt-foundry --resource-group $RG \
  --name route-to-nva \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address "$FW_IP"


az network vnet subnet update \
  --name snet-foundry-pe --vnet-name $VNET \
  --resource-group $RG --route-table rt-foundry

az network vnet subnet update \
  --name snet-services-pe --vnet-name $VNET \
  --resource-group $RG --route-table rt-foundry

# Step 4 — Connected Services

# Storage
az storage account create \
  --name $STORAGE --resource-group $RG \
  --location $LOCATION --sku Standard_LRS \
  --allow-blob-public-access false

az storage container create \
  --name agentfiles --account-name $STORAGE \
  --auth-mode login

# CosmosDB
az cosmosdb create \
  --name $COSMOS --resource-group $RG \
  --locations regionName=$LOCATION \
  --capabilities EnableServerless

az cosmosdb sql database create \
  --account-name $COSMOS --resource-group $RG \
  --name agent-threads

# AI Search
az search service create \
  --name $SEARCH --resource-group $RG \
  --location $LOCATION --sku basic

# Key Vault
# sometimes it may take time to delete the last key vault
az keyvault create \
  --name $KV --resource-group $RG --location $LOCATION

# Step 5 — Private Endpoints for Connected Services

SUBNET_SVC=$(az network vnet subnet show \
  --name snet-services-pe --vnet-name $VNET \
  --resource-group $RG --query id -o tsv)

SUBNET_SVC=$(echo "$SUBNET_SVC" | tr -d '\r')

STORAGE_ID=$(az storage account show \
  --name $STORAGE -g $RG --query id -o tsv)

STORAGE_ID=$(echo "$STORAGE_ID" | tr -d '\r')

COSMOS_ID=$(az cosmosdb show \
  --name $COSMOS -g $RG --query id -o tsv)

COSMOS_ID=$(echo "$COSMOS_ID" | tr -d '\r')

SEARCH_ID=$(az search service show \
  --name $SEARCH -g $RG --query id -o tsv)

SEARCH_ID=$(echo "$SEARCH_ID" | tr -d '\r')

KV_ID=$(az keyvault show \
  --name $KV -g $RG --query id -o tsv)
KV_ID=$(echo "$KV_ID" | tr -d '\r')

az network private-endpoint create \
  --name pe-storage -g $RG --location $LOCATION \
  --subnet "$SUBNET_SVC" \
  --private-connection-resource-id "$STORAGE_ID" \
  --group-id blob --connection-name conn-storage

az network private-endpoint create \
  --name pe-cosmos -g $RG --location $LOCATION \
  --subnet "$SUBNET_SVC" \
  --private-connection-resource-id "$COSMOS_ID" \
  --group-id Sql --connection-name conn-cosmos

az network private-endpoint create \
  --name pe-search -g $RG --location $LOCATION \
  --subnet "$SUBNET_SVC" \
  --private-connection-resource-id "$SEARCH_ID" \
  --group-id searchService --connection-name conn-search

az network private-endpoint create \
  --name pe-kv -g $RG --location $LOCATION \
  --subnet "$SUBNET_SVC" \
  --private-connection-resource-id "$KV_ID" \
  --group-id vault --connection-name conn-kv


# Step 6 — Private DNS Zones

for ZONE in \
  "privatelink.cognitiveservices.azure.com" \
  "privatelink.openai.azure.com" \
  "privatelink.services.ai.azure.com" \
  "privatelink.blob.core.windows.net" \
  "privatelink.documents.azure.com" \
  "privatelink.search.windows.net" \
  "privatelink.vaultcore.azure.net"; do

  az network private-dns zone create \
    --resource-group $RG --name $ZONE

  az network private-dns link vnet create \
    --resource-group $RG --zone-name $ZONE \
    --name "link-$(echo $ZONE | tr '.' '-')" \
    --virtual-network $VNET \
    --registration-enabled false
done

az network private-endpoint dns-zone-group create \
  -g $RG --endpoint-name pe-storage --name zg-storage \
  --private-dns-zone "privatelink.blob.core.windows.net" \
  --zone-name zone1

az network private-endpoint dns-zone-group create \
  -g $RG --endpoint-name pe-cosmos --name zg-cosmos \
  --private-dns-zone "privatelink.documents.azure.com" \
  --zone-name zone1

az network private-endpoint dns-zone-group create \
  -g $RG --endpoint-name pe-search --name zg-search \
  --private-dns-zone "privatelink.search.windows.net" \
  --zone-name zone1

az network private-endpoint dns-zone-group create \
  -g $RG --endpoint-name pe-kv --name zg-kv \
  --private-dns-zone "privatelink.vaultcore.azure.net" \
  --zone-name zone1

# Step 7 — Create Foundry Account (with networkInjections at creation time)
## Critical lesson learned: networkInjections must be set at creation time. It cannot be updated after. Use snet-agent-2 (clean subnet).
AGENT_SUBNET_ID=$(az network vnet subnet show \
  --name snet-agent-2 --vnet-name $VNET \
  --resource-group $RG --query id -o tsv)

AGENT_SUBNET_ID=$(echo "$AGENT_SUBNET_ID" | tr -d '\r')

python3 << EOF
import subprocess
import json
import time

SUB = "$SUB"
RG = "$RG"
LOCATION = "$LOCATION"
ACCOUNT = "$ACCOUNT"
subnet = "$AGENT_SUBNET_ID"

body = json.dumps({
    "location": LOCATION,
    "kind": "AIServices",
    "sku": {"name": "S0"},
    "identity": {"type": "SystemAssigned"},
    "properties": {
        "customSubDomainName": ACCOUNT,
        "publicNetworkAccess": "Disabled",
        "allowProjectManagement": True,
        "networkInjections": [{
            "scenario": "Agent",
            "subnetArmId": subnet,
            "useMicrosoftManagedNetwork": False
        }]
    }
})

result = subprocess.run(
    ["az", "rest", "--method", "PUT",
     "--url", f"https://management.azure.com/subscriptions/{SUB}/resourceGroups/{RG}/providers/Microsoft.CognitiveServices/accounts/{ACCOUNT}?api-version=2025-10-01-preview",
     "--headers", "Content-Type=application/json",
     "--body", body],
    capture_output=True, text=True
)
print("Return code:", result.returncode)
print("STDOUT:", result.stdout[:500] if result.stdout else "EMPTY")
print("STDERR:", result.stderr[:300] if result.stderr else "EMPTY")

print("\nPolling provisioning state...")
for i in range(15):
    time.sleep(20)
    poll = subprocess.run(
        ["az", "cognitiveservices", "account", "show",
         "--name", ACCOUNT, "--resource-group", RG,
         "--query", "{state:properties.provisioningState, networkInjections:properties.networkInjections}",
         "--output", "json"],
        capture_output=True, text=True
    )
    print(f"Attempt {i+1}: {poll.stdout.strip()[:300]}")
    if "Succeeded" in poll.stdout:
        print("✔ Foundry account ready!")
        break
EOF

# Step 8 — Foundry Private Endpoint

FOUNDRY_ID=$(az cognitiveservices account show \
  --name $ACCOUNT -g $RG --query id -o tsv)
FOUNDRY_ID=$(echo "$FOUNDRY_ID" | tr -d '\r')

SUBNET_PE=$(az network vnet subnet show \
  --name snet-foundry-pe --vnet-name $VNET \
  --resource-group $RG --query id -o tsv)
SUBNET_PE=$(echo "$SUBNET_PE" | tr -d '\r')

az network private-endpoint create \
  --name pe-storage -g $RG --location $LOCATION \
  --subnet "$SUBNET_SVC" \
  --private-connection-resource-id "$STORAGE_ID" \
  --group-id blob --connection-name conn-storage

az network private-endpoint create \
  --name pe-cosmos -g $RG --location $LOCATION \
  --subnet "$SUBNET_SVC" \
  --private-connection-resource-id "$COSMOS_ID" \
  --group-id Sql --connection-name conn-cosmos

az network private-endpoint create \
  --name pe-search -g $RG --location $LOCATION \
  --subnet "$SUBNET_SVC" \
  --private-connection-resource-id "$SEARCH_ID" \
  --group-id searchService --connection-name conn-search

az network private-endpoint create \
  --name pe-kv -g $RG --location $LOCATION \
  --subnet "$SUBNET_SVC" \
  --private-connection-resource-id "$KV_ID" \
  --group-id vault --connection-name conn-kv

# Step 9 : Create Project
az cognitiveservices account project create \
  --name $ACCOUNT --resource-group $RG \
  --project-name $PROJECT --location $LOCATION

# Step 10 : Create RBAC

PROJECT_MI=$(az rest --method GET \
  --url "https://management.azure.com/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.CognitiveServices/accounts/$ACCOUNT/projects/$PROJECT?api-version=2025-06-01" \
  --query "identity.principalId" -o tsv)

PROJECT_MI=$(echo "$PROJECT_MI" | tr -d '\r')

echo "Project MI: $PROJECT_MI"

STORAGE_ID=$(az storage account show --name $STORAGE -g $RG --query id -o tsv)
SEARCH_ID=$(az search service show --name $SEARCH -g $RG --query id -o tsv)
STORAGE_ID=$(echo "$STORAGE_ID" | tr -d '\r')
SEARCH_ID=$(echo "$SEARCH_ID" | tr -d '\r')

az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee "$PROJECT_MI" --scope "$STORAGE_ID"

az cosmosdb sql role assignment create \
  --resource-group $RG --account-name $COSMOS \
  --role-definition-id "00000000-0000-0000-0000-000000000002" \
  --principal-id "$PROJECT_MI" --scope "/"

az role assignment create \
  --role "Search Index Data Contributor" \
  --assignee "$PROJECT_MI" --scope "$SEARCH_ID"

az role assignment create \
  --role "Search Service Contributor" \
  --assignee "$PROJECT_MI" --scope "$SEARCH_ID"

# Step 11 — Project Connections

python3 << EOF
import subprocess
import json

SUB = "$SUB"
RG = "$RG"
ACCOUNT = "$ACCOUNT"
PROJECT = "$PROJECT"
STORAGE = "$STORAGE"
COSMOS = "$COSMOS"
SEARCH = "$SEARCH"

base = f"https://management.azure.com/subscriptions/{SUB}/resourceGroups/{RG}/providers/Microsoft.CognitiveServices/accounts/{ACCOUNT}/projects/{PROJECT}/connections"

connections = [
    {
        "name": "shared-cosmosdb-connection",
        "body": {
            "properties": {
                "category": "CosmosDb",
                "target": f"https://{COSMOS}.documents.azure.com:443/",
                "authType": "AAD"
            }
        }
    },
    {
        "name": "shared-search-connection",
        "body": {
            "properties": {
                "category": "CognitiveSearch",
                "target": f"https://{SEARCH}.search.windows.net/",
                "authType": "AAD"
            }
        }
    },
    {
        "name": "shared-storage-connection",
        "body": {
            "properties": {
                "category": "AzureBlob",
                "target": f"https://{STORAGE}.blob.core.windows.net/agentfiles",
                "authType": "AAD",
                "metadata": {
                    "ContainerName": "agentfiles",
                    "AccountName": STORAGE
                }
            }
        }
    }
]

for conn in connections:
    url = f"{base}/{conn['name']}?api-version=2025-06-01"
    result = subprocess.run(
        ["az", "rest", "--method", "PUT", "--url", url,
         "--headers", "Content-Type=application/json",
         "--body", json.dumps(conn["body"])],
        capture_output=True, text=True
    )
    status = "✔ OK" if result.returncode == 0 else "✘ FAILED"
    print(f"{status} - {conn['name']}")
    if result.returncode != 0:
        print(f"  Error: {result.stderr[:200]}")
EOF




```

