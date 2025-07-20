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

## 5. Create Azure Resource Group

Create the resource group that will contain all your private Container App resources.


```bash
# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

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


```bash
# Create subnet for Container Apps (requires minimum /21 prefix)
az network vnet subnet create \
  --name $SUBNET_NAME \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --address-prefixes 10.0.0.0/21 \
  --delegations 'Microsoft.App/environments'
```


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

## 11. Configure Private DNS

Set up private DNS zone and VNet linking for internal name resolution.


```bash
# Create a private DNS zone for Container Apps
az network private-dns zone create \
  --name $PRIVATE_DNS_ZONE \
  --resource-group $RESOURCE_GROUP
```


```bash
# Link the private DNS zone to the VNet
az network private-dns link vnet create \
  --name "container-apps-dns-link" \
  --resource-group $RESOURCE_GROUP \
  --zone-name $PRIVATE_DNS_ZONE \
  --virtual-network $VNET_NAME \
  --registration-enabled false
```

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


```bash
# Get the Container Apps Environment resource ID
CONTAINER_APP_ENV_ID=$(az containerapp env show \
  --name $CONTAINER_APP_ENV \
  --resource-group $RESOURCE_GROUP \
  --query id \
  --output tsv)
echo "CONTAINER_APP_ENV_ID=$CONTAINER_APP_ENV_ID"
```


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


```bash
# Create DNS record for the private endpoint
az network private-dns record-set a create \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --zone-name $PRIVATE_DNS_ZONE
```


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

## 14. Test Private Access

Use the jumper container app to test connectivity to your private Container App.


```bash
# Output variables block for shell testing
echo "======== VARS FOR SHELL ========="
echo "CONTAINER_APP_INTERNAL_URL=$CONTAINER_APP_INTERNAL_URL"
echo "PRIVATE_ENDPOINT_IP=$PRIVATE_ENDPOINT_IP"
echo "=================================="

echo "======== COMMANDS FOR SHELL ====="
echo "curl -k \$CONTAINER_APP_INTERNAL_URL"
echo "================================="
```


```bash
# Test connectivity using interactive shell (run this in a separate terminal)
# This will open an interactive session in the jumper container
az containerapp exec \
  --name $JUMPER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
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
