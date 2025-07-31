# Private Azure Container Apps Deployment Playbook

This interactive notebook guides you through deploying a private Azure Container Apps service with VNet integration using Azure CLI commands. You can run each cell individually or execute them in sequence to complete the full private network deployment.

## Overview

This notebook will help you:
- Install Azure CLI prerequisites
- Authenticate with Azure
- Create a Virtual Network with proper subnet configuration
- Deploy a private Container Apps Environment with VNet integration
- Create a private Container App with internal ingress
- Set up a jumper Container App for testing private connectivity
- Configure private endpoints and DNS
- Test private network access
- Clean up resources when done

**Note:** Make sure you have appropriate permissions in your Azure subscription to create networking resources, private endpoints, and Container Apps.

## 1. Install Azure CLI Prerequisites

First, check that the Azure CLI and containerapps extension is installed correctly.

If running in a devcontainer, this should all already be configured


```bash
# Check Azure CLI Prerequisites
echo "🔍 Checking Azure CLI prerequisites..."
echo ""

# Check if Azure CLI is installed
if command -v az &> /dev/null; then
    echo "✅ Azure CLI is installed: $(az version --query '"azure-cli"' -o tsv)"
else
    echo "❌ ERROR: Azure CLI is not installed!"
    echo ""
    echo "📋 To install Azure CLI:"
    echo "  • On macOS: brew install azure-cli"
    echo "  • On Ubuntu/Debian: curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash"
    echo "  • On Windows: winget install Microsoft.AzureCLI"
    echo "  • Or visit: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli"
    echo ""
    exit 1
fi

# Check if Container Apps extension is installed
if az extension list --query "[?name=='containerapp'].name" -o tsv | grep -q "containerapp"; then
    echo "✅ Container Apps extension is installed"
else
    echo "⚠️  WARNING: Container Apps extension is not installed!"
    echo ""
    echo "📋 To install the extension, run:"
    echo "  az extension add --name containerapp --upgrade"
    echo ""
fi

echo ""
echo "🎯 Prerequisites check complete!"
```

    🔍 Checking Azure CLI prerequisites...
    
    
    ✅ Azure CLI is installed: 2.75.0
    ✅ Azure CLI is installed: 2.75.0
    ✅ Container Apps extension is installed
    ✅ Container Apps extension is installed
    
    
    🎯 Prerequisites check complete!
    🎯 Prerequisites check complete!


## 2. Configure Azure Authentication

Login to Azure and verify your subscription context.


```bash
# Login to Azure (only if not already logged in)
if ! az account show >/dev/null 2>&1; then
  echo "Not logged in to Azure. Logging in... (sub selection disabled)"
  az config set core.login_experience_v2=off
  az login
else
  echo "Already logged in to Azure."
fi
```

    Already logged in to Azure.


## 3. Set Deployment Variables

Define the configuration variables for your private Container App deployment. You can modify these values as needed for your specific deployment.


```bash
# Set variables for private setup with unique resource group name
export LOCATION="uksouth"
export RESOURCE_GROUP="private-container-apps-rg"
export VNET_NAME="private-vnet"
export SUBNET_NAME="container-apps-subnet"
export CONTAINER_APP_ENV="private-container-app-env"
export CONTAINER_APP_NAME="private-hello-world-app"
export JUMPER_APP_NAME="jumper-container-app"
export PRIVATE_ENDPOINT_NAME="container-app-private-endpoint"
export PRIVATE_DNS_ZONE="privatelink.azurecontainerapps.io"

# Display the variables
echo "Resource Group:     $RESOURCE_GROUP"
echo "Location:           $LOCATION"
echo "Virtual Network:    $VNET_NAME"
echo "Subnet Name:        $SUBNET_NAME"
echo "Container App Env:  $CONTAINER_APP_ENV"
echo "Container App Name: $CONTAINER_APP_NAME"
echo "Jumper App Name:    $JUMPER_APP_NAME"
echo "Private Endpoint:   $PRIVATE_ENDPOINT_NAME"
echo "Private DNS Zone:   $PRIVATE_DNS_ZONE"
```

    Resource Group:     private-container-apps-rg
    Location:           uksouth
    Location:           uksouth
    Virtual Network:    private-vnet
    Virtual Network:    private-vnet
    Subnet Name:        container-apps-subnet
    Subnet Name:        container-apps-subnet
    Container App Env:  private-container-app-env
    Container App Env:  private-container-app-env
    Container App Name: private-hello-world-app
    Container App Name: private-hello-world-app
    Jumper App Name:    jumper-container-app
    Jumper App Name:    jumper-container-app
    Private Endpoint:   container-app-private-endpoint
    Private Endpoint:   container-app-private-endpoint
    Private DNS Zone:   privatelink.azurecontainerapps.io
    Private DNS Zone:   privatelink.azurecontainerapps.io


## 4. Register Resource Providers


```bash
# Register resource providers
echo "🔄 Registering resource providers..."

az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.Network

echo "⏳ Waiting for resource providers to be registered..."

# Wait for Microsoft.OperationalInsights to be registered
while [ "$(az provider show --namespace Microsoft.OperationalInsights --query registrationState -o tsv)" != "Registered" ]; do
    echo "  • Microsoft.OperationalInsights: $(az provider show --namespace Microsoft.OperationalInsights --query registrationState -o tsv)"
    sleep 10
done

# Wait for Microsoft.Network to be registered
while [ "$(az provider show --namespace Microsoft.Network --query registrationState -o tsv)" != "Registered" ]; do
    echo "  • Microsoft.Network: $(az provider show --namespace Microsoft.Network --query registrationState -o tsv)"
    sleep 10
done

echo "✅ All resource providers are now registered!"
echo "  • Microsoft.OperationalInsights: Registered"
echo "  • Microsoft.Network: Registered"
```

    🔄 Registering resource providers...
    ⏳ Waiting for resource providers to be registered...
    ⏳ Waiting for resource providers to be registered...
    ✅ All resource providers are now registered!
    ✅ All resource providers are now registered!
      • Microsoft.OperationalInsights: Registered
      • Microsoft.OperationalInsights: Registered
      • Microsoft.Network: Registered
      • Microsoft.Network: Registered


## 5. Create Azure Resource Group

Create the resource group that will contain all your private Container App resources.


```bash
# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

    {
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg",
      "location": "uksouth",
      "managedBy": null,
      "name": "private-container-apps-rg",
      "properties": {
        "provisioningState": "Succeeded"
      },
      "tags": null,
      "type": "Microsoft.Resources/resourceGroups"
    }
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg",
      "location": "uksouth",
      "managedBy": null,
      "name": "private-container-apps-rg",
      "properties": {
        "provisioningState": "Succeeded"
      },
      "tags": null,
      "type": "Microsoft.Resources/resourceGroups"
    }


## 6. Create Virtual Network and Subnets

Create the Virtual Network with the required subnet configuration for Container Apps.


```bash
# Create Virtual Network
az network vnet create \
  --name $VNET_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --address-prefixes 10.0.0.0/16
```

    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:50: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    [K{- Finished ..
      "newVNet": {
        "addressSpace": {
          "addressPrefixes": [
            "10.0.0.0/16"
          ]
        },
        "enableDdosProtection": false,
        "etag": "W/\"7fe73ecb-9e0d-467b-abfc-2c72687268ca\"",
        "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/virtualNetworks/private-vnet",
        "location": "uksouth",
        "name": "private-vnet",
        "privateEndpointVNetPolicies": "Disabled",
        "provisioningState": "Succeeded",
        "resourceGroup": "private-container-apps-rg",
        "resourceGuid": "07d1a234-072d-4f52-8d06-842cb0ace721",
        "subnets": [],
        "type": "Microsoft.Network/virtualNetworks",
        "virtualNetworkPeerings": []
      }
    }
    [K{- Finished ..
      "newVNet": {
        "addressSpace": {
          "addressPrefixes": [
            "10.0.0.0/16"
          ]
        },
        "enableDdosProtection": false,
        "etag": "W/\"7fe73ecb-9e0d-467b-abfc-2c72687268ca\"",
        "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/virtualNetworks/private-vnet",
        "location": "uksouth",
        "name": "private-vnet",
        "privateEndpointVNetPolicies": "Disabled",
        "provisioningState": "Succeeded",
        "resourceGroup": "private-container-apps-rg",
        "resourceGuid": "07d1a234-072d-4f52-8d06-842cb0ace721",
        "subnets": [],
        "type": "Microsoft.Network/virtualNetworks",
        "virtualNetworkPeerings": []
      }
    }



```bash
# Create subnet for Container Apps (requires minimum /21 prefix)
az network vnet subnet create \
  --name $SUBNET_NAME \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --address-prefixes 10.0.0.0/21 \
  --delegations 'Microsoft.App/environments'
```

    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:50: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    [K{| Finished ..
      "addressPrefix": "10.0.0.0/21",
      "delegations": [
        {
          "actions": [
            "Microsoft.Network/virtualNetworks/subnets/join/action"
          ],
          "etag": "W/\"f593a016-61d5-4eed-8b23-0ff6f4784a72\"",
          "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/virtualNetworks/private-vnet/subnets/container-apps-subnet/delegations/0",
          "name": "0",
          "provisioningState": "Succeeded",
          "resourceGroup": "private-container-apps-rg",
          "serviceName": "Microsoft.App/environments",
          "type": "Microsoft.Network/virtualNetworks/subnets/delegations"
        }
      ],
      "etag": "W/\"f593a016-61d5-4eed-8b23-0ff6f4784a72\"",
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/virtualNetworks/private-vnet/subnets/container-apps-subnet",
      "name": "container-apps-subnet",
      "privateEndpointNetworkPolicies": "Disabled",
      "privateLinkServiceNetworkPolicies": "Enabled",
      "provisioningState": "Succeeded",
      "resourceGroup": "private-container-apps-rg",
      "type": "Microsoft.Network/virtualNetworks/subnets"
    }
    [K{| Finished ..
      "addressPrefix": "10.0.0.0/21",
      "delegations": [
        {
          "actions": [
            "Microsoft.Network/virtualNetworks/subnets/join/action"
          ],
          "etag": "W/\"f593a016-61d5-4eed-8b23-0ff6f4784a72\"",
          "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/virtualNetworks/private-vnet/subnets/container-apps-subnet/delegations/0",
          "name": "0",
          "provisioningState": "Succeeded",
          "resourceGroup": "private-container-apps-rg",
          "serviceName": "Microsoft.App/environments",
          "type": "Microsoft.Network/virtualNetworks/subnets/delegations"
        }
      ],
      "etag": "W/\"f593a016-61d5-4eed-8b23-0ff6f4784a72\"",
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/virtualNetworks/private-vnet/subnets/container-apps-subnet",
      "name": "container-apps-subnet",
      "privateEndpointNetworkPolicies": "Disabled",
      "privateLinkServiceNetworkPolicies": "Enabled",
      "provisioningState": "Succeeded",
      "resourceGroup": "private-container-apps-rg",
      "type": "Microsoft.Network/virtualNetworks/subnets"
    }



```bash
# Get subnet ID for Container Apps Environment
SUBNET_ID=$(az network vnet subnet show \
  --name $SUBNET_NAME \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --query id \
  --output tsv)
echo "SUBNET_ID=$SUBNET_ID"
```

    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:50: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    SUBNET_ID=/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/virtualNetworks/private-vnet/subnets/container-apps-subnet
    SUBNET_ID=/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/virtualNetworks/private-vnet/subnets/container-apps-subnet


## 7. Create Private Container Apps Environment

Create the Container Apps environment with VNet integration and internal-only access.


```bash
# Create the Container Apps Environment with VNet integration
az containerapp env create \
  --name $CONTAINER_APP_ENV \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --infrastructure-subnet-resource-id $SUBNET_ID \
  --internal-only true
```

    [93mThe behavior of this command has been altered by the following extension: containerapp[0m
    [93mNo Log Analytics workspace provided.[0m
    [93mNo Log Analytics workspace provided.[0m
    [93mGenerating a Log Analytics workspace with name "workspace-privatecontainerappsrgbDQS"[0m
    [93mGenerating a Log Analytics workspace with name "workspace-privatecontainerappsrgbDQS"[0m
    [K[93mg ..ed ....
    Container Apps environment created. To deploy a container app, use: az containerapp create --help
    [0m
    {
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/managedEnvironments/private-container-app-env",
      "location": "UK South",
      "name": "private-container-app-env",
      "properties": {
        "appInsightsConfiguration": null,
        "appLogsConfiguration": {
          "destination": "log-analytics",
          "logAnalyticsConfiguration": {
            "customerId": "6ea17eea-56a7-499f-a275-ab1fe94ec3a5",
            "dynamicJsonColumns": false,
            "sharedKey": null
          }
        },
        "availabilityZones": null,
    [K[93m
    Container Apps environment created. To deploy a container app, use: az containerapp create --help
    [0m
    {
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/managedEnvironments/private-container-app-env",
      "location": "UK South",
      "name": "private-container-app-env",
      "properties": {
        "appInsightsConfiguration": null,
        "appLogsConfiguration": {
          "destination": "log-analytics",
          "logAnalyticsConfiguration": {
            "customerId": "6ea17eea-56a7-499f-a275-ab1fe94ec3a5",
            "dynamicJsonColumns": false,
            "sharedKey": null
          }
        },
        "availabilityZones": null,
        "customDomainConfiguration": {
          "certificateKeyVaultProperties": null,
          "certificatePassword": null,
          "certificateValue": null,
          "customDomainVerificationId": "AA389110BB6F0D9646DB70682842B7E3D1E3E7B59E9D6D4B5CEB528CB637AF89",
          "dnsSuffix": null,
          "expirationDate": null,
          "subjectName": null,
          "thumbprint": null
        },
        "daprAIConnectionString": null,
        "daprAIInstrumentationKey": null,
        "daprConfiguration": {
          "version": "1.13.6-msft.4"
        },
        "defaultDomain": "calmwater-75a475e8.uksouth.azurecontainerapps.io",
        "diskEncryptionConfiguration": null,
        "eventStreamEndpoint": "https://uksouth.azurecontainerapps.dev/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/managedEnvironments/private-container-app-env/eventstream",
        "infrastructureResourceGroup": "ME_private-container-app-env_private-container-apps-rg_uksouth",
        "ingressConfiguration": null,
        "kedaConfiguration": {
          "version": "2.16.1"
        },
        "openTelemetryConfiguration": null,
        "peerAuthentication": {
          "mtls": {
            "enabled": false
          }
        },
        "peerTrafficConfiguration": {
          "encryption": {
            "enabled": false
          }
        },
        "provisioningState": "Succeeded",
        "publicNetworkAccess": "Disabled",
        "staticIp": "10.0.5.30",
        "vnetConfiguration": {
          "dockerBridgeCidr": null,
          "infrastructureSubnetId": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/virtualNetworks/private-vnet/subnets/container-apps-subnet",
          "internal": true,
          "platformReservedCidr": null,
          "platformReservedDnsIP": null
        },
        "workloadProfiles": [
          {
            "enableFips": false,
            "name": "Consumption",
            "workloadProfileType": "Consumption"
          }
        ],
        "zoneRedundant": false
      },
      "resourceGroup": "private-container-apps-rg",
      "systemData": {
        "createdAt": "2025-07-29T09:24:17.8258631",
        "createdBy": "messenger@pmgledhill.com",
        "createdByType": "User",
        "customDomainConfiguration": {
          "certificateKeyVaultProperties": null,
          "certificatePassword": null,
          "certificateValue": null,
          "customDomainVerificationId": "AA389110BB6F0D9646DB70682842B7E3D1E3E7B59E9D6D4B5CEB528CB637AF89",
          "dnsSuffix": null,
          "expirationDate": null,
          "subjectName": null,
          "thumbprint": null
        },
        "daprAIConnectionString": null,
        "daprAIInstrumentationKey": null,
        "daprConfiguration": {
          "version": "1.13.6-msft.4"
        },
        "defaultDomain": "calmwater-75a475e8.uksouth.azurecontainerapps.io",
        "diskEncryptionConfiguration": null,
        "eventStreamEndpoint": "https://uksouth.azurecontainerapps.dev/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/managedEnvironments/private-container-app-env/eventstream",
        "infrastructureResourceGroup": "ME_private-container-app-env_private-container-apps-rg_uksouth",
        "ingressConfiguration": null,
        "kedaConfiguration": {
          "version": "2.16.1"
        },
        "openTelemetryConfiguration": null,
        "peerAuthentication": {
          "mtls": {
            "enabled": false
          }
        },
        "peerTrafficConfiguration": {
          "encryption": {
            "enabled": false
          }
        },
        "provisioningState": "Succeeded",
        "publicNetworkAccess": "Disabled",
        "staticIp": "10.0.5.30",
        "vnetConfiguration": {
          "dockerBridgeCidr": null,
          "infrastructureSubnetId": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/virtualNetworks/private-vnet/subnets/container-apps-subnet",
          "internal": true,
          "platformReservedCidr": null,
          "platformReservedDnsIP": null
        },
        "workloadProfiles": [
          {
            "enableFips": false,
            "name": "Consumption",
            "workloadProfileType": "Consumption"
          }
        ],
        "zoneRedundant": false
      },
      "resourceGroup": "private-container-apps-rg",
      "systemData": {
        "createdAt": "2025-07-29T09:24:17.8258631",
        "createdBy": "messenger@pmgledhill.com",
        "createdByType": "User",
        "lastModifiedAt": "2025-07-29T09:24:17.8258631",
        "lastModifiedBy": "messenger@pmgledhill.com",
        "lastModifiedByType": "User"
      },
      "type": "Microsoft.App/managedEnvironments"
    }
        "lastModifiedAt": "2025-07-29T09:24:17.8258631",
        "lastModifiedBy": "messenger@pmgledhill.com",
        "lastModifiedByType": "User"
      },
      "type": "Microsoft.App/managedEnvironments"
    }


## 8. Deploy Private Container App

Deploy the hello-world container application with internal ingress only.


```bash
# Create the Container App (internal only)
az containerapp create \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINER_APP_ENV \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --target-port 80 \
  --ingress internal \
  --min-replicas 0 \
  --max-replicas 5 \
  --cpu 0.25 \
  --memory 0.5Gi
```

    [93mThe behavior of this command has been altered by the following extension: containerapp[0m
    [K[93mg ..
    Container app created. Access your app at https://private-hello-world-app.internal.calmwater-75a475e8.uksouth.azurecontainerapps.io/
    [0m
    {
    [K[93m
    Container app created. Access your app at https://private-hello-world-app.internal.calmwater-75a475e8.uksouth.azurecontainerapps.io/
    [0m
    {
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/containerapps/private-hello-world-app",
      "identity": {
        "type": "None"
      },
      "location": "UK South",
      "name": "private-hello-world-app",
      "properties": {
        "configuration": {
          "activeRevisionsMode": "Single",
          "dapr": null,
          "identitySettings": [],
          "ingress": {
            "additionalPortMappings": null,
            "allowInsecure": false,
            "clientCertificateMode": null,
            "corsPolicy": null,
            "customDomains": null,
            "exposedPort": 0,
            "external": false,
            "fqdn": "private-hello-world-app.internal.calmwater-75a475e8.uksouth.azurecontainerapps.io",
            "ipSecurityRestrictions": null,
            "stickySessions": null,
            "targetPort": 80,
            "targetPortHttpScheme": null,
            "traffic": [
              {
                "latestRevision": true,
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/containerapps/private-hello-world-app",
      "identity": {
        "type": "None"
      },
      "location": "UK South",
      "name": "private-hello-world-app",
      "properties": {
        "configuration": {
          "activeRevisionsMode": "Single",
          "dapr": null,
          "identitySettings": [],
          "ingress": {
            "additionalPortMappings": null,
            "allowInsecure": false,
            "clientCertificateMode": null,
            "corsPolicy": null,
            "customDomains": null,
            "exposedPort": 0,
            "external": false,
            "fqdn": "private-hello-world-app.internal.calmwater-75a475e8.uksouth.azurecontainerapps.io",
            "ipSecurityRestrictions": null,
            "stickySessions": null,
            "targetPort": 80,
            "targetPortHttpScheme": null,
            "traffic": [
              {
                "latestRevision": true,
                "weight": 100
              }
            ],
            "transport": "Auto"
          },
          "maxInactiveRevisions": 100,
          "registries": null,
          "revisionTransitionThreshold": null,
          "runtime": null,
          "secrets": null,
          "service": null,
          "targetLabel": ""
        },
        "customDomainVerificationId": "AA389110BB6F0D9646DB70682842B7E3D1E3E7B59E9D6D4B5CEB528CB637AF89",
        "delegatedIdentities": [],
        "environmentId": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/managedEnvironments/private-container-app-env",
        "eventStreamEndpoint": "https://uksouth.azurecontainerapps.dev/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/containerApps/private-hello-world-app/eventstream",
        "latestReadyRevisionName": "private-hello-world-app--nyynnp6",
        "latestRevisionFqdn": "private-hello-world-app--nyynnp6.internal.calmwater-75a475e8.uksouth.azurecontainerapps.io",
        "latestRevisionName": "private-hello-world-app--nyynnp6",
        "managedEnvironmentId": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/managedEnvironments/private-container-app-env",
        "outboundIpAddresses": [
          "20.90.238.99",
          "20.90.239.180",
          "85.210.163.187",
          "74.177.240.109",
          "172.166.80.179",
          "131.145.9.241",
          "85.210.166.17",
          "85.210.48.136",
          "74.177.120.102",
          "131.145.48.93",
          "85.210.163.208",
          "74.177.193.62",
          "20.108.130.227",
          "20.108.130.191",
                "weight": 100
              }
            ],
            "transport": "Auto"
          },
          "maxInactiveRevisions": 100,
          "registries": null,
          "revisionTransitionThreshold": null,
          "runtime": null,
          "secrets": null,
          "service": null,
          "targetLabel": ""
        },
        "customDomainVerificationId": "AA389110BB6F0D9646DB70682842B7E3D1E3E7B59E9D6D4B5CEB528CB637AF89",
        "delegatedIdentities": [],
        "environmentId": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/managedEnvironments/private-container-app-env",
        "eventStreamEndpoint": "https://uksouth.azurecontainerapps.dev/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/containerApps/private-hello-world-app/eventstream",
        "latestReadyRevisionName": "private-hello-world-app--nyynnp6",
        "latestRevisionFqdn": "private-hello-world-app--nyynnp6.internal.calmwater-75a475e8.uksouth.azurecontainerapps.io",
        "latestRevisionName": "private-hello-world-app--nyynnp6",
        "managedEnvironmentId": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/managedEnvironments/private-container-app-env",
        "outboundIpAddresses": [
          "20.90.238.99",
          "20.90.239.180",
          "85.210.163.187",
          "74.177.240.109",
          "172.166.80.179",
          "131.145.9.241",
          "85.210.166.17",
          "85.210.48.136",
          "74.177.120.102",
          "131.145.48.93",
          "85.210.163.208",
          "74.177.193.62",
          "20.108.130.227",
          "20.108.130.191",
          "85.210.25.6",
          "85.210.24.221",
          "4.158.34.249",
          "20.162.138.167",
          "4.158.113.223",
          "4.159.3.89",
          "20.108.216.9",
          "20.108.217.125",
          "74.177.201.70",
          "131.145.91.209",
          "4.158.91.74",
          "4.158.89.64",
          "131.145.40.6",
          "74.177.198.189",
          "85.210.223.160",
          "74.177.245.93",
          "74.177.198.181",
          "131.145.77.209",
          "20.108.124.221",
          "74.177.162.123",
          "20.26.109.122",
          "131.145.124.57",
          "85.210.222.64",
          "20.26.109.180",
          "74.177.141.127",
          "85.210.223.56",
          "74.177.245.142",
          "131.145.83.126",
          "131.145.99.107",
          "20.108.125.221",
          "4.159.25.18",
          "4.158.92.46",
          "20.162.145.125",
          "4.158.143.149",
          "4.158.143.92",
          "4.250.234.44",
          "51.142.70.93"
        ],
        "patchingMode": "Automatic",
        "provisioningState": "Succeeded",
        "runningStatus": "Running",
        "template": {
          "containers": [
            {
              "image": "mcr.microsoft.com/azuredocs/containerapps-helloworld:latest",
              "imageType": "ContainerImage",
              "name": "private-hello-world-app",
              "resources": {
                "cpu": 0.25,
                "ephemeralStorage": "1Gi",
                "memory": "0.5Gi"
              }
            }
          ],
          "initContainers": null,
          "revisionSuffix": "",
          "scale": {
            "cooldownPeriod": 300,
            "maxReplicas": 5,
            "minReplicas": 0,
            "pollingInterval": 30,
            "rules": null
          },
          "serviceBinds": null,
          "terminationGracePeriodSeconds": null,
          "volumes": null
        },
        "workloadProfileName": "Consumption"
      },
      "resourceGroup": "private-container-apps-rg",
      "systemData": {
        "createdAt": "2025-07-29T09:28:11.6625922",
        "createdBy": "messenger@pmgledhill.com",
        "createdByType": "User",
        "lastModifiedAt": "2025-07-29T09:28:11.6625922",
        "lastModifiedBy": "messenger@pmgledhill.com",
        "lastModifiedByType": "User"
      },
      "type": "Microsoft.App/containerApps"
    }
          "85.210.25.6",
          "85.210.24.221",
          "4.158.34.249",
          "20.162.138.167",
          "4.158.113.223",
          "4.159.3.89",
          "20.108.216.9",
          "20.108.217.125",
          "74.177.201.70",
          "131.145.91.209",
          "4.158.91.74",
          "4.158.89.64",
          "131.145.40.6",
          "74.177.198.189",
          "85.210.223.160",
          "74.177.245.93",
          "74.177.198.181",
          "131.145.77.209",
          "20.108.124.221",
          "74.177.162.123",
          "20.26.109.122",
          "131.145.124.57",
          "85.210.222.64",
          "20.26.109.180",
          "74.177.141.127",
          "85.210.223.56",
          "74.177.245.142",
          "131.145.83.126",
          "131.145.99.107",
          "20.108.125.221",
          "4.159.25.18",
          "4.158.92.46",
          "20.162.145.125",
          "4.158.143.149",
          "4.158.143.92",
          "4.250.234.44",
          "51.142.70.93"
        ],
        "patchingMode": "Automatic",
        "provisioningState": "Succeeded",
        "runningStatus": "Running",
        "template": {
          "containers": [
            {
              "image": "mcr.microsoft.com/azuredocs/containerapps-helloworld:latest",
              "imageType": "ContainerImage",
              "name": "private-hello-world-app",
              "resources": {
                "cpu": 0.25,
                "ephemeralStorage": "1Gi",
                "memory": "0.5Gi"
              }
            }
          ],
          "initContainers": null,
          "revisionSuffix": "",
          "scale": {
            "cooldownPeriod": 300,
            "maxReplicas": 5,
            "minReplicas": 0,
            "pollingInterval": 30,
            "rules": null
          },
          "serviceBinds": null,
          "terminationGracePeriodSeconds": null,
          "volumes": null
        },
        "workloadProfileName": "Consumption"
      },
      "resourceGroup": "private-container-apps-rg",
      "systemData": {
        "createdAt": "2025-07-29T09:28:11.6625922",
        "createdBy": "messenger@pmgledhill.com",
        "createdByType": "User",
        "lastModifiedAt": "2025-07-29T09:28:11.6625922",
        "lastModifiedBy": "messenger@pmgledhill.com",
        "lastModifiedByType": "User"
      },
      "type": "Microsoft.App/containerApps"
    }


## 9. Get Container App Internal URL

Retrieve the internal URL of your deployed private container application.


```bash
# Get the Container App internal URL
CONTAINER_APP_INTERNAL_URL=$(az containerapp show \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query properties.configuration.ingress.fqdn \
  --output tsv)

echo "Container App Internal URL: https://$CONTAINER_APP_INTERNAL_URL"
```

    WARNING: The behavior of this command has been altered by the following extension: containerapp
    Container App Internal URL: https://private-hello-world-app.internal.calmwater-75a475e8.uksouth.azurecontainerapps.io
    Container App Internal URL: https://private-hello-world-app.internal.calmwater-75a475e8.uksouth.azurecontainerapps.io


## 10. Create Jumper Container App for Testing

Create a jumper container app that can be used to test connectivity to the private container app.


```bash
# Create YAML configuration file for the Jumper Container App
cat > jumper-app-config.yaml << EOF
location: uksouth
type: Microsoft.App/containerApps
name: $JUMPER_APP_NAME
properties:
  configuration:
    activeRevisionsMode: Single
  environmentId: /subscriptions/$(az account show --query id --output tsv)/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.App/managedEnvironments/$CONTAINER_APP_ENV
  template:
    containers:
    - name: jumper-container
      image: mcr.microsoft.com/azure-cli:latest
      command:
      - /bin/bash
      args:
      - -c
      - timeout 600 bash -c 'while true; do sleep 30; done' && echo 'Jumper container shutting down after 10 minutes'
      resources:
        cpu: 0.25
        memory: 0.5Gi
      env:
      - name: APP_URL
        value: https://$CONTAINER_APP_INTERNAL_URL
    scale:
      minReplicas: 0
      maxReplicas: 1
EOF
```


```bash
# Create the Jumper Container App using the YAML configuration
az containerapp create \
  --name $JUMPER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --yaml jumper-app-config.yaml
```

    [93mThe behavior of this command has been altered by the following extension: containerapp[0m
    [K[93mg ..
    Container app created. To access it over HTTPS, enable ingress: az containerapp ingress enable -n jumper-container-app -g private-container-apps-rg --type external --target-port <port> --transport auto
    [0m
    {
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/containerapps/jumper-container-app",
      "identity": {
        "type": "None"
      },
      "location": "UK South",
      "name": "jumper-container-app",
      "properties": {
        "configuration": {
          "activeRevisionsMode": "Single",
          "dapr": null,
          "identitySettings": [],
    [K[93m
    Container app created. To access it over HTTPS, enable ingress: az containerapp ingress enable -n jumper-container-app -g private-container-apps-rg --type external --target-port <port> --transport auto
    [0m
    {
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/containerapps/jumper-container-app",
      "identity": {
        "type": "None"
      },
      "location": "UK South",
      "name": "jumper-container-app",
      "properties": {
        "configuration": {
          "activeRevisionsMode": "Single",
          "dapr": null,
          "identitySettings": [],
          "ingress": null,
          "maxInactiveRevisions": 100,
          "ingress": null,
          "maxInactiveRevisions": 100,
          "registries": null,
          "revisionTransitionThreshold": null,
          "runtime": null,
          "secrets": null,
          "service": null,
          "targetLabel": ""
        },
        "customDomainVerificationId": "AA389110BB6F0D9646DB70682842B7E3D1E3E7B59E9D6D4B5CEB528CB637AF89",
        "delegatedIdentities": [],
        "environmentId": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/managedEnvironments/private-container-app-env",
        "eventStreamEndpoint": "https://uksouth.azurecontainerapps.dev/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/containerApps/jumper-container-app/eventstream",
        "latestReadyRevisionName": "jumper-container-app--4i2k2fs",
        "latestRevisionFqdn": "",
        "latestRevisionName": "jumper-container-app--4i2k2fs",
        "managedEnvironmentId": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/managedEnvironments/private-container-app-env",
        "outboundIpAddresses": [
          "20.90.238.99",
          "20.90.239.180",
          "85.210.163.187",
          "74.177.240.109",
          "172.166.80.179",
          "131.145.9.241",
          "85.210.166.17",
          "85.210.48.136",
          "74.177.120.102",
          "131.145.48.93",
          "85.210.163.208",
          "74.177.193.62",
          "registries": null,
          "revisionTransitionThreshold": null,
          "runtime": null,
          "secrets": null,
          "service": null,
          "targetLabel": ""
        },
        "customDomainVerificationId": "AA389110BB6F0D9646DB70682842B7E3D1E3E7B59E9D6D4B5CEB528CB637AF89",
        "delegatedIdentities": [],
        "environmentId": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/managedEnvironments/private-container-app-env",
        "eventStreamEndpoint": "https://uksouth.azurecontainerapps.dev/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/containerApps/jumper-container-app/eventstream",
        "latestReadyRevisionName": "jumper-container-app--4i2k2fs",
        "latestRevisionFqdn": "",
        "latestRevisionName": "jumper-container-app--4i2k2fs",
        "managedEnvironmentId": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/managedEnvironments/private-container-app-env",
        "outboundIpAddresses": [
          "20.90.238.99",
          "20.90.239.180",
          "85.210.163.187",
          "74.177.240.109",
          "172.166.80.179",
          "131.145.9.241",
          "85.210.166.17",
          "85.210.48.136",
          "74.177.120.102",
          "131.145.48.93",
          "85.210.163.208",
          "74.177.193.62",
          "20.108.130.227",
          "20.108.130.191",
          "85.210.25.6",
          "85.210.24.221",
          "4.158.34.249",
          "20.162.138.167",
          "4.158.113.223",
          "4.159.3.89",
          "20.108.216.9",
          "20.108.217.125",
          "74.177.201.70",
          "131.145.91.209",
          "4.158.91.74",
          "4.158.89.64",
          "131.145.40.6",
          "74.177.198.189",
          "85.210.223.160",
          "74.177.245.93",
          "74.177.198.181",
          "131.145.77.209",
          "20.108.124.221",
          "74.177.162.123",
          "20.26.109.122",
          "131.145.124.57",
          "85.210.222.64",
          "20.26.109.180",
          "74.177.141.127",
          "85.210.223.56",
          "74.177.245.142",
          "131.145.83.126",
          "131.145.99.107",
          "20.108.125.221",
          "4.159.25.18",
          "4.158.92.46",
          "20.162.145.125",
          "4.158.143.149",
          "4.158.143.92",
          "4.250.234.44",
          "51.142.70.93"
        ],
        "patchingMode": "Automatic",
        "provisioningState": "Succeeded",
        "runningStatus": "Running",
        "template": {
          "containers": [
            {
              "args": [
                "-c",
                "timeout 600 bash -c 'while true; do sleep 30; done' && echo 'Jumper container shutting down after 10 minutes'"
              ],
              "command": [
                "/bin/bash"
              ],
              "env": [
                {
                  "name": "APP_URL"
                }
              ],
              "image": "mcr.microsoft.com/azure-cli:latest",
              "imageType": "ContainerImage",
              "name": "jumper-container",
              "resources": {
                "cpu": 0.25,
                "ephemeralStorage": "1Gi",
                "memory": "0.5Gi"
              }
            }
          ],
          "initContainers": null,
          "revisionSuffix": "",
          "scale": {
            "cooldownPeriod": 300,
            "maxReplicas": 1,
            "minReplicas": 0,
            "pollingInterval": 30,
            "rules": null
          },
          "serviceBinds": null,
          "terminationGracePeriodSeconds": null,
          "volumes": null
        },
        "workloadProfileName": "Consumption"
      },
      "resourceGroup": "private-container-apps-rg",
      "systemData": {
        "createdAt": "2025-07-29T09:28:46.9369358",
        "createdBy": "messenger@pmgledhill.com",
        "createdByType": "User",
        "lastModifiedAt": "2025-07-29T09:28:46.9369358",
        "lastModifiedBy": "messenger@pmgledhill.com",
        "lastModifiedByType": "User"
      },
      "type": "Microsoft.App/containerApps"
    }
          "20.108.130.227",
          "20.108.130.191",
          "85.210.25.6",
          "85.210.24.221",
          "4.158.34.249",
          "20.162.138.167",
          "4.158.113.223",
          "4.159.3.89",
          "20.108.216.9",
          "20.108.217.125",
          "74.177.201.70",
          "131.145.91.209",
          "4.158.91.74",
          "4.158.89.64",
          "131.145.40.6",
          "74.177.198.189",
          "85.210.223.160",
          "74.177.245.93",
          "74.177.198.181",
          "131.145.77.209",
          "20.108.124.221",
          "74.177.162.123",
          "20.26.109.122",
          "131.145.124.57",
          "85.210.222.64",
          "20.26.109.180",
          "74.177.141.127",
          "85.210.223.56",
          "74.177.245.142",
          "131.145.83.126",
          "131.145.99.107",
          "20.108.125.221",
          "4.159.25.18",
          "4.158.92.46",
          "20.162.145.125",
          "4.158.143.149",
          "4.158.143.92",
          "4.250.234.44",
          "51.142.70.93"
        ],
        "patchingMode": "Automatic",
        "provisioningState": "Succeeded",
        "runningStatus": "Running",
        "template": {
          "containers": [
            {
              "args": [
                "-c",
                "timeout 600 bash -c 'while true; do sleep 30; done' && echo 'Jumper container shutting down after 10 minutes'"
              ],
              "command": [
                "/bin/bash"
              ],
              "env": [
                {
                  "name": "APP_URL"
                }
              ],
              "image": "mcr.microsoft.com/azure-cli:latest",
              "imageType": "ContainerImage",
              "name": "jumper-container",
              "resources": {
                "cpu": 0.25,
                "ephemeralStorage": "1Gi",
                "memory": "0.5Gi"
              }
            }
          ],
          "initContainers": null,
          "revisionSuffix": "",
          "scale": {
            "cooldownPeriod": 300,
            "maxReplicas": 1,
            "minReplicas": 0,
            "pollingInterval": 30,
            "rules": null
          },
          "serviceBinds": null,
          "terminationGracePeriodSeconds": null,
          "volumes": null
        },
        "workloadProfileName": "Consumption"
      },
      "resourceGroup": "private-container-apps-rg",
      "systemData": {
        "createdAt": "2025-07-29T09:28:46.9369358",
        "createdBy": "messenger@pmgledhill.com",
        "createdByType": "User",
        "lastModifiedAt": "2025-07-29T09:28:46.9369358",
        "lastModifiedBy": "messenger@pmgledhill.com",
        "lastModifiedByType": "User"
      },
      "type": "Microsoft.App/containerApps"
    }


## 11. Configure Private DNS

Set up private DNS zone and VNet linking for internal name resolution.


```bash
# Create a private DNS zone for Container Apps
az network private-dns zone create \
  --name $PRIVATE_DNS_ZONE \
  --resource-group $RESOURCE_GROUP
```

    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:50: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    [K{\ Finished ..
      "etag": "8214b435-3150-467f-bf4b-6289a71fd08f",
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/privateDnsZones/privatelink.azurecontainerapps.io",
      "location": "global",
      "maxNumberOfRecordSets": 25000,
      "maxNumberOfVirtualNetworkLinks": 1000,
      "maxNumberOfVirtualNetworkLinksWithRegistration": 100,
      "name": "privatelink.azurecontainerapps.io",
      "numberOfRecordSets": 1,
      "numberOfVirtualNetworkLinks": 0,
      "numberOfVirtualNetworkLinksWithRegistration": 0,
      "provisioningState": "Succeeded",
      "resourceGroup": "private-container-apps-rg",
      "type": "Microsoft.Network/privateDnsZones"
    }
    [K{\ Finished ..
      "etag": "8214b435-3150-467f-bf4b-6289a71fd08f",
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/privateDnsZones/privatelink.azurecontainerapps.io",
      "location": "global",
      "maxNumberOfRecordSets": 25000,
      "maxNumberOfVirtualNetworkLinks": 1000,
      "maxNumberOfVirtualNetworkLinksWithRegistration": 100,
      "name": "privatelink.azurecontainerapps.io",
      "numberOfRecordSets": 1,
      "numberOfVirtualNetworkLinks": 0,
      "numberOfVirtualNetworkLinksWithRegistration": 0,
      "provisioningState": "Succeeded",
      "resourceGroup": "private-container-apps-rg",
      "type": "Microsoft.Network/privateDnsZones"
    }



```bash
# Link the private DNS zone to the VNet
az network private-dns link vnet create \
  --name "container-apps-dns-link" \
  --resource-group $RESOURCE_GROUP \
  --zone-name $PRIVATE_DNS_ZONE \
  --virtual-network $VNET_NAME \
  --registration-enabled false
```

    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:50: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    [K{\ Finished ..
      "etag": "\"0800f172-0000-0100-0000-688894910000\"",
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/privateDnsZones/privatelink.azurecontainerapps.io/virtualNetworkLinks/container-apps-dns-link",
      "location": "global",
      "name": "container-apps-dns-link",
      "provisioningState": "Succeeded",
      "registrationEnabled": false,
      "resolutionPolicy": "Default",
      "resourceGroup": "private-container-apps-rg",
      "type": "Microsoft.Network/privateDnsZones/virtualNetworkLinks",
      "virtualNetwork": {
        "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/virtualNetworks/private-vnet",
        "resourceGroup": "private-container-apps-rg"
      },
      "virtualNetworkLinkState": "Completed"
    }
    [K{\ Finished ..
      "etag": "\"0800f172-0000-0100-0000-688894910000\"",
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/privateDnsZones/privatelink.azurecontainerapps.io/virtualNetworkLinks/container-apps-dns-link",
      "location": "global",
      "name": "container-apps-dns-link",
      "provisioningState": "Succeeded",
      "registrationEnabled": false,
      "resolutionPolicy": "Default",
      "resourceGroup": "private-container-apps-rg",
      "type": "Microsoft.Network/privateDnsZones/virtualNetworkLinks",
      "virtualNetwork": {
        "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/virtualNetworks/private-vnet",
        "resourceGroup": "private-container-apps-rg"
      },
      "virtualNetworkLinkState": "Completed"
    }


## 12. Create Private Endpoint

Set up private endpoint for secure access to the Container Apps Environment.


```bash
# Create a private endpoint subnet (separate from container apps subnet)
az network vnet subnet create \
  --name "private-endpoint-subnet" \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --address-prefixes 10.0.8.0/24
```

    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:50: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    [K{/ Finished ..
      "addressPrefix": "10.0.8.0/24",
      "delegations": [],
      "etag": "W/\"60003cd9-e2da-4b9f-b5ac-257da5d2896c\"",
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/virtualNetworks/private-vnet/subnets/private-endpoint-subnet",
      "name": "private-endpoint-subnet",
      "privateEndpointNetworkPolicies": "Disabled",
      "privateLinkServiceNetworkPolicies": "Enabled",
      "provisioningState": "Succeeded",
      "resourceGroup": "private-container-apps-rg",
      "type": "Microsoft.Network/virtualNetworks/subnets"
    }
    [K{/ Finished ..
      "addressPrefix": "10.0.8.0/24",
      "delegations": [],
      "etag": "W/\"60003cd9-e2da-4b9f-b5ac-257da5d2896c\"",
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/virtualNetworks/private-vnet/subnets/private-endpoint-subnet",
      "name": "private-endpoint-subnet",
      "privateEndpointNetworkPolicies": "Disabled",
      "privateLinkServiceNetworkPolicies": "Enabled",
      "provisioningState": "Succeeded",
      "resourceGroup": "private-container-apps-rg",
      "type": "Microsoft.Network/virtualNetworks/subnets"
    }



```bash
# Get the Container Apps Environment resource ID
CONTAINER_APP_ENV_ID=$(az containerapp env show \
  --name $CONTAINER_APP_ENV \
  --resource-group $RESOURCE_GROUP \
  --query id \
  --output tsv)
echo "CONTAINER_APP_ENV_ID=$CONTAINER_APP_ENV_ID"
```

    WARNING: The behavior of this command has been altered by the following extension: containerapp
    CONTAINER_APP_ENV_ID=/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/managedEnvironments/private-container-app-env
    CONTAINER_APP_ENV_ID=/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/managedEnvironments/private-container-app-env



```bash
# Create private endpoint for the Container Apps Environment
az network private-endpoint create \
  --name $PRIVATE_ENDPOINT_NAME \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --subnet "private-endpoint-subnet" \
  --private-connection-resource-id $CONTAINER_APP_ENV_ID \
  --group-id managedEnvironments \
  --connection-name "container-app-connection"
```

    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:50: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    [K{\ Finished ..
      "customDnsConfigs": [
        {
          "fqdn": "*.calmwater-75a475e8.uksouth.azurecontainerapps.io",
          "ipAddresses": [
            "10.0.8.4"
          ]
        }
      ],
      "customNetworkInterfaceName": "",
      "etag": "W/\"864cc4df-7896-42be-b30e-823e441a9103\"",
    [K{\ Finished ..
      "customDnsConfigs": [
        {
          "fqdn": "*.calmwater-75a475e8.uksouth.azurecontainerapps.io",
          "ipAddresses": [
            "10.0.8.4"
          ]
        }
      ],
      "customNetworkInterfaceName": "",
      "etag": "W/\"864cc4df-7896-42be-b30e-823e441a9103\"",
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/privateEndpoints/container-app-private-endpoint",
      "ipConfigurations": [],
      "location": "uksouth",
      "manualPrivateLinkServiceConnections": [],
      "name": "container-app-private-endpoint",
      "networkInterfaces": [
        {
          "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/networkInterfaces/container-app-private-endpoint.nic.44455b79-91e0-46db-bee9-97bc725cd4a1",
          "resourceGroup": "private-container-apps-rg"
        }
      ],
      "privateLinkServiceConnections": [
        {
          "etag": "W/\"864cc4df-7896-42be-b30e-823e441a9103\"",
          "groupIds": [
            "managedEnvironments"
          ],
          "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/privateEndpoints/container-app-private-endpoint/privateLinkServiceConnections/container-app-connection",
          "name": "container-app-connection",
          "privateLinkServiceConnectionState": {
            "actionsRequired": "None",
            "description": "Auto-approved",
            "status": "Approved"
          },
          "privateLinkServiceId": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/managedEnvironments/private-container-app-env",
          "provisioningState": "Succeeded",
          "resourceGroup": "private-container-apps-rg",
          "type": "Microsoft.Network/privateEndpoints/privateLinkServiceConnections"
        }
      ],
      "provisioningState": "Succeeded",
      "resourceGroup": "private-container-apps-rg",
      "subnet": {
        "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/virtualNetworks/private-vnet/subnets/private-endpoint-subnet",
        "resourceGroup": "private-container-apps-rg"
      },
      "type": "Microsoft.Network/privateEndpoints"
    }
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/privateEndpoints/container-app-private-endpoint",
      "ipConfigurations": [],
      "location": "uksouth",
      "manualPrivateLinkServiceConnections": [],
      "name": "container-app-private-endpoint",
      "networkInterfaces": [
        {
          "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/networkInterfaces/container-app-private-endpoint.nic.44455b79-91e0-46db-bee9-97bc725cd4a1",
          "resourceGroup": "private-container-apps-rg"
        }
      ],
      "privateLinkServiceConnections": [
        {
          "etag": "W/\"864cc4df-7896-42be-b30e-823e441a9103\"",
          "groupIds": [
            "managedEnvironments"
          ],
          "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/privateEndpoints/container-app-private-endpoint/privateLinkServiceConnections/container-app-connection",
          "name": "container-app-connection",
          "privateLinkServiceConnectionState": {
            "actionsRequired": "None",
            "description": "Auto-approved",
            "status": "Approved"
          },
          "privateLinkServiceId": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.App/managedEnvironments/private-container-app-env",
          "provisioningState": "Succeeded",
          "resourceGroup": "private-container-apps-rg",
          "type": "Microsoft.Network/privateEndpoints/privateLinkServiceConnections"
        }
      ],
      "provisioningState": "Succeeded",
      "resourceGroup": "private-container-apps-rg",
      "subnet": {
        "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/virtualNetworks/private-vnet/subnets/private-endpoint-subnet",
        "resourceGroup": "private-container-apps-rg"
      },
      "type": "Microsoft.Network/privateEndpoints"
    }


## 13. Configure Private DNS Records

Set up DNS records for the private endpoint to enable name resolution.


```bash
# Get the private endpoint's private IP
PRIVATE_ENDPOINT_IP=$(az network private-endpoint show \
  --name $PRIVATE_ENDPOINT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query 'customDnsConfigs[0].ipAddresses[0]' \
  --output tsv)
echo "PRIVATE_ENDPOINT_IP=$PRIVATE_ENDPOINT_IP"
```

    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:50: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    PRIVATE_ENDPOINT_IP=10.0.8.4
    PRIVATE_ENDPOINT_IP=10.0.8.4



```bash
# Create DNS record for the private endpoint
az network private-dns record-set a create \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --zone-name $PRIVATE_DNS_ZONE
```

    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:50: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    {
      "aRecords": [],
      "etag": "19e2b374-be13-4790-9dc8-048887e7e42b",
      "fqdn": "private-hello-world-app.privatelink.azurecontainerapps.io.",
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/privateDnsZones/privatelink.azurecontainerapps.io/A/private-hello-world-app",
      "isAutoRegistered": false,
      "name": "private-hello-world-app",
      "resourceGroup": "private-container-apps-rg",
      "ttl": 3600,
      "type": "Microsoft.Network/privateDnsZones/A"
    }
    {
      "aRecords": [],
      "etag": "19e2b374-be13-4790-9dc8-048887e7e42b",
      "fqdn": "private-hello-world-app.privatelink.azurecontainerapps.io.",
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/privateDnsZones/privatelink.azurecontainerapps.io/A/private-hello-world-app",
      "isAutoRegistered": false,
      "name": "private-hello-world-app",
      "resourceGroup": "private-container-apps-rg",
      "ttl": 3600,
      "type": "Microsoft.Network/privateDnsZones/A"
    }



```bash
# Add A record with private endpoint IP
az network private-dns record-set a add-record \
  --record-set-name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --zone-name $PRIVATE_DNS_ZONE \
  --ipv4-address $PRIVATE_ENDPOINT_IP

echo "Private endpoint created with IP: $PRIVATE_ENDPOINT_IP"
echo "Container App is accessible privately at: https://$CONTAINER_APP_INTERNAL_URL"
```

    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:50: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:51: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.group_kwargs['preview_info'] = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:132: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      if self.AZ_PREVIEW_INFO:
    /usr/local/lib/python3.13/site-packages/azure/cli/core/aaz/_command.py:133: FutureWarning: functools.partial will be a method descriptor in future Python versions; wrap it in staticmethod() if you want to preserve the old behavior
      self.preview_info = self.AZ_PREVIEW_INFO(cli_ctx=self.cli_ctx)
    {
      "aRecords": [
        {
          "ipv4Address": "10.0.8.4"
        }
      ],
      "etag": "c1160edd-0544-45ce-99ce-c87e81eb86b1",
      "fqdn": "private-hello-world-app.privatelink.azurecontainerapps.io.",
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/privateDnsZones/privatelink.azurecontainerapps.io/A/private-hello-world-app",
      "isAutoRegistered": false,
      "name": "private-hello-world-app",
      "resourceGroup": "private-container-apps-rg",
      "ttl": 3600,
      "type": "Microsoft.Network/privateDnsZones/A"
    }
    {
      "aRecords": [
        {
          "ipv4Address": "10.0.8.4"
        }
      ],
      "etag": "c1160edd-0544-45ce-99ce-c87e81eb86b1",
      "fqdn": "private-hello-world-app.privatelink.azurecontainerapps.io.",
      "id": "/subscriptions/77cf2ee7-d234-424d-99d2-2227e6ae2dd2/resourceGroups/private-container-apps-rg/providers/Microsoft.Network/privateDnsZones/privatelink.azurecontainerapps.io/A/private-hello-world-app",
      "isAutoRegistered": false,
      "name": "private-hello-world-app",
      "resourceGroup": "private-container-apps-rg",
      "ttl": 3600,
      "type": "Microsoft.Network/privateDnsZones/A"
    }
    Private endpoint created with IP: 10.0.8.4
    Private endpoint created with IP: 10.0.8.4
    Container App is accessible privately at: https://private-hello-world-app.internal.calmwater-75a475e8.uksouth.azurecontainerapps.io
    Container App is accessible privately at: https://private-hello-world-app.internal.calmwater-75a475e8.uksouth.azurecontainerapps.io


## 14. Test Private Access

Use the jumper container app to test connectivity to your private Container App.


```bash
# Output variables block for shell testing
echo "======== VARS FOR SHELL ========="
echo "JUMPER_APP_NAME=$JUMPER_APP_NAME"
echo "RESOURCE_GROUP=$RESOURCE_GROUP"
echo "CONTAINER_APP_INTERNAL_URL=$CONTAINER_APP_INTERNAL_URL"
echo "PRIVATE_ENDPOINT_IP=$PRIVATE_ENDPOINT_IP"
echo "=================================="

echo "======== COMMANDS FOR SHELL ====="
echo "curl -k \$CONTAINER_APP_INTERNAL_URL"
echo "================================="
```

    ======== VARS FOR SHELL =========
    JUMPER_APP_NAME=jumper-container-app
    JUMPER_APP_NAME=jumper-container-app
    RESOURCE_GROUP=private-container-apps-rg
    RESOURCE_GROUP=private-container-apps-rg
    CONTAINER_APP_INTERNAL_URL=private-hello-world-app.internal.calmwater-75a475e8.uksouth.azurecontainerapps.io
    CONTAINER_APP_INTERNAL_URL=private-hello-world-app.internal.calmwater-75a475e8.uksouth.azurecontainerapps.io
    PRIVATE_ENDPOINT_IP=10.0.8.4
    PRIVATE_ENDPOINT_IP=10.0.8.4
    ==================================
    ==================================
    ======== COMMANDS FOR SHELL =====
    ======== COMMANDS FOR SHELL =====
    curl -k $CONTAINER_APP_INTERNAL_URL
    curl -k $CONTAINER_APP_INTERNAL_URL
    =================================
    =================================



```bash
# Test connectivity using interactive shell (run this in a separate terminal)
# This will open an interactive session in the jumper container
az containerapp exec \
  --name $v \
  --command "/bin/sh"
```

## 15. Verification Commands

Run these commands to verify your deployment status and configuration.


```bash
# Check Container Apps Environment status
az containerapp env show \
  --name $CONTAINER_APP_ENV \
  --resource-group $RESOURCE_GROUP \
  --query "{name:name, location:location, vnetConfiguration:vnetConfiguration}" \
  --output table
```


```bash
# Check Container App status
az containerapp show \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "{name:name, provisioningState:properties.provisioningState, fqdn:properties.configuration.ingress.fqdn}" \
  --output table
```


```bash
# Check Jumper Container App status
az containerapp show \
  --name $JUMPER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "{name:name, provisioningState:properties.provisioningState, replicas:properties.template.scale}" \
  --output table
```


```bash
# List private endpoints
az network private-endpoint list \
  --resource-group $RESOURCE_GROUP \
  --query "[].{name:name, provisioningState:provisioningState, privateEndpointIp:customDnsConfigs[0].ipAddresses[0]}" \
  --output table
```


```bash
# Note: Automated testing via az containerapp exec with multiple arguments
# is not supported in the same way as kubectl exec. 
# Use the interactive shell method in section 16 instead.

echo "⚠️  Automated connectivity testing is not available via az containerapp exec"
echo "🔧 Use the interactive shell method described in section 16 to test connectivity"
echo "💡 Or run verification commands in section 15 to check deployment status"
```


```bash
# At this point, your private Container Apps environment is fully deployed
# Continue to the next section to test connectivity manually
exit 1
echo "✅ Private Container Apps deployment is complete!"
echo "🔄 Proceed to section 16 to test connectivity using the jumper container"
```

## 16. Play with Container

Now you've a container app deployed. You can exec into the 'jumper' container, and start
running commands against it.

To exec into the jumper run this:

```sh
az containerapp exec \
   --name $JUMPER_APP_NAME \
   --resource-group $RESOURCE_GROUP \
   --command "/bin/bash"
```

And then to make a CURL against the app container, you can use the handy `APP_URL` env var:

```sh
curl $APP_URL
```

## 17. Cleanup Resources

When you're done with your private Container App setup, use the following commands to clean up the resources to avoid ongoing charges.

**⚠️ Warning:** These commands will permanently delete your resources. Make sure you no longer need them before proceeding.


```bash
# Delete the entire resource group (this will delete all resources)
# ⚠️ This will delete ALL resources in the resource group!
az group delete \
  --name $RESOURCE_GROUP \
  --yes --no-wait
```


```bash
# Clean up local files
rm -f jumper-app-config.yaml
```

## Summary

🎉 **Congratulations!** You have successfully:

1. ✅ Installed Azure CLI prerequisites
2. ✅ Authenticated with Azure
3. ✅ Set deployment variables
4. ✅ Registered required resource providers
5. ✅ Created a resource group
6. ✅ Created a virtual network with proper subnet configuration
7. ✅ Created a private Container Apps environment with VNet integration
8. ✅ Deployed a private containerized application with internal ingress
9. ✅ Retrieved the internal URL of your container app
10. ✅ Created a jumper container app for testing
11. ✅ Configured private DNS zone and VNet linking
12. ✅ Set up private endpoints for secure access
13. ✅ Configured DNS records for name resolution
14. ✅ Tested private connectivity
15. ✅ Verified deployment status
16. ✅ Tested manual connectivity using the jumper container
17. ✅ (Optional) Cleaned up resources

## Key Differences from Public Setup

- **VNet Integration**: Container Apps Environment created with `--internal-only true` and linked to a VNet subnet
- **Private Ingress**: Container App uses `--ingress internal` instead of `--ingress external`
- **Private Endpoints**: Created to provide private connectivity to the Container Apps Environment
- **Private DNS**: Custom DNS zone for internal name resolution
- **Jumper Container App**: A second Container App in the same VNet for testing private connectivity
- **Scale-to-Zero Architecture**: Both apps can scale to zero when not in use, eliminating compute costs
- **Container-to-Container Communication**: Uses native VNet connectivity between container apps
- **No VM Dependencies**: Fully containerized testing approach without virtual machines

## Cost Considerations

**Daily Cost Estimate: ~$1.81 USD** (with no traffic)

Key cost components:
- **Private Endpoint**: $0.36/day ($10.80/month)
- **Private DNS Zone**: $1.20/day ($15/month)
- **Container Apps**: $0.00 when scaled to zero (pay only when processing requests)
- **Jumper App**: $0.00 when idle (only costs during testing sessions)

## Next Steps

- **Deploy your own container**: Replace the hello-world image with your own containerized application
- **Configure custom domains**: Set up custom domain names for your private Container Apps
- **Implement CI/CD**: Set up automated deployments using GitHub Actions or Azure DevOps
- **Add monitoring**: Configure Application Insights and Log Analytics for monitoring
- **Scale testing**: Experiment with different replica counts and resource allocations
- **Network security**: Add Network Security Groups and additional security layers

For more advanced Container Apps features and private networking scenarios, check out the [Azure Container Apps documentation](https://docs.microsoft.com/azure/container-apps/).
