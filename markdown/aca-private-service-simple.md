# Simple Private Azure Container Apps Deployment Playbook

This interactive notebook guides you through deploying a private Azure Container Apps service with VNet integration using Azure CLI commands. This simplified version skips private endpoints and DNS zones since apps in the same environment can communicate directly. This will also reduces cost.

## Overview

This notebook will help you:
- Install Azure CLI prerequisites
- Authenticate with Azure
- Create a Virtual Network with proper subnet configuration
- Deploy a private Container Apps Environment with VNet integration
- Create a private Container App with internal ingress
- Set up a jumper Container App for testing connectivity
- Test internal app-to-app communication
- Clean up resources when done

**Note:** Apps in the same Container Apps environment can communicate using the internal DNS pattern: `https://<app-name>.internal.<env-name>.azurecontainerapps.io`

## 1. Install Azure CLI Prerequisites

First, check that the Azure CLI and containerapps extension is installed correctly.

If running in a devcontainer, this should all already be configured


```python
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


```python
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

Define the configuration variables for your private Container App deployment. This simplified version doesn't need private endpoint or DNS zone variables.


```python
# Set variables for simplified private setup
export LOCATION="uksouth"
export RESOURCE_GROUP="simple-private-container-apps-rg"
export VNET_NAME="simple-private-vnet"
export SUBNET_NAME="container-apps-subnet"
export CONTAINER_APP_ENV="simple-private-container-app-env"
export CONTAINER_APP_NAME="simple-private-hello-world-app"
export JUMPER_APP_NAME="simple-jumper-container-app"

# Display the variables
echo "Resource Group:     $RESOURCE_GROUP"
echo "Location:           $LOCATION"
echo "Virtual Network:    $VNET_NAME"
echo "Subnet Name:        $SUBNET_NAME"
echo "Container App Env:  $CONTAINER_APP_ENV"
echo "Container App Name: $CONTAINER_APP_NAME"
echo "Jumper App Name:    $JUMPER_APP_NAME"
```

## 4. Register Resource Providers


```python
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


```python
# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

## 6. Create Virtual Network and Subnets

Create the Virtual Network with the required subnet configuration for Container Apps.


```python
# Create Virtual Network
az network vnet create \
  --name $VNET_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --address-prefixes 10.0.0.0/16
```


```python
# Create subnet for Container Apps (requires minimum /21 prefix)
az network vnet subnet create \
  --name $SUBNET_NAME \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --address-prefixes 10.0.0.0/21 \
  --delegations 'Microsoft.App/environments'
```


```python
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


```python
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


```python
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


```python
# Get the Container App internal URL
CONTAINER_APP_INTERNAL_URL=$(az containerapp show \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query properties.configuration.ingress.fqdn \
  --output tsv)

echo "Container App Internal URL: https://$CONTAINER_APP_INTERNAL_URL"
echo "Alternative Internal URL: https://$CONTAINER_APP_NAME.internal.$CONTAINER_APP_ENV.azurecontainerapps.io"
```

## 10. Create Jumper Container App for Testing

Create a jumper container app that can be used to test connectivity to the private container app. Since both apps are in the same environment, they can communicate directly.


```python
# Create the Jumper Container App with environment variables for testing
az containerapp create \
  --name $JUMPER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINER_APP_ENV \
  --image mcr.microsoft.com/azure-cli:latest \
  --command "/bin/bash" \
  --args "-c" "timeout 600 bash -c 'while true; do sleep 30; done' && echo 'Jumper container shutting down after 10 minutes'" \
  --env-vars "APP_URL=https://$CONTAINER_APP_INTERNAL_URL" "ALT_APP_URL=https://$CONTAINER_APP_NAME.internal.$CONTAINER_APP_ENV.azurecontainerapps.io" \
  --min-replicas 0 \
  --max-replicas 1 \
  --cpu 0.25 \
  --memory 0.5Gi
```

## 11. Test Internal Connectivity

Use the jumper container app to test connectivity to your private Container App using the internal DNS.


```python
# Output variables for shell testing
echo "======== VARS FOR SHELL ========="
echo "CONTAINER_APP_INTERNAL_URL=$CONTAINER_APP_INTERNAL_URL"
echo "ALTERNATIVE_URL=https://$CONTAINER_APP_NAME.internal.$CONTAINER_APP_ENV.azurecontainerapps.io"
echo "================================="

echo "======== COMMANDS FOR SHELL ====="
echo "# Test using the provided FQDN:"
echo "curl -k \$APP_URL"
echo ""
echo "# Test using the alternative internal pattern:"
echo "curl -k \$ALT_APP_URL"
echo "================================="
```

## 12. Interactive Testing

Open an interactive shell in the jumper container to test connectivity manually.


```python
# Test connectivity using interactive shell (run this in a separate terminal)
# This will open an interactive session in the jumper container
az containerapp exec \
  --name $JUMPER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --command "/bin/bash"
```

## 13. Verification Commands

Run these commands to verify your deployment status and configuration.


```python
# Check Container Apps Environment status
az containerapp env show \
  --name $CONTAINER_APP_ENV \
  --resource-group $RESOURCE_GROUP \
  --query "{name:name, location:location, vnetConfiguration:vnetConfiguration}" \
  --output table
```


```python
# Check Container App status
az containerapp show \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "{name:name, provisioningState:properties.provisioningState, fqdn:properties.configuration.ingress.fqdn}" \
  --output table
```


```python
# Check Jumper Container App status
az containerapp show \
  --name $JUMPER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "{name:name, provisioningState:properties.provisioningState, replicas:properties.template.scale}" \
  --output table
```

## 14. Testing Guide

Now you have a simplified private container app setup. You can exec into the 'jumper' container and test connectivity.

To exec into the jumper run:

```bash
az containerapp exec \
   --name $JUMPER_APP_NAME \
   --resource-group $RESOURCE_GROUP \
   --command "/bin/bash"
```

Then to make a CURL against the app container, you can use either:

```bash
# Using the provided environment variable:
curl $APP_URL

# Using the alternative internal DNS pattern:
curl $ALT_APP_URL

# Or manually specify the URL:
curl https://simple-private-hello-world-app.internal.simple-private-container-app-env.azurecontainerapps.io
```

**Key Benefits of this simplified approach:**
- No private endpoints needed for app-to-app communication
- No private DNS zones required
- Lower cost and complexity
- Still maintains network isolation with VNet integration
- Apps can communicate using built-in internal DNS

## 15. Cleanup Resources

When you're done with your private Container App setup, use the following commands to clean up the resources to avoid ongoing charges.

**⚠️ Warning:** These commands will permanently delete your resources. Make sure you no longer need them before proceeding.


```python
# Delete the entire resource group (this will delete all resources)
# ⚠️ This will delete ALL resources in the resource group!
az group delete \
  --name $RESOURCE_GROUP \
  --yes --no-wait
```

## Summary

🎉 **Congratulations!** You have successfully created a simplified private Container Apps setup:

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
11. ✅ Tested internal app-to-app connectivity
12. ✅ Verified deployment status
13. ✅ (Optional) Cleaned up resources

## Key Simplifications vs Full Private Setup

- **No Private Endpoints**: Apps in the same environment can communicate directly
- **No Private DNS Zones**: Uses built-in internal DNS resolution
- **Lower Cost**: Eliminates ~$26/month in private endpoint and DNS zone costs
- **Simpler Architecture**: Fewer components to manage and troubleshoot
- **Same Security**: Still maintains VNet isolation and internal-only access

## Cost Comparison

**Simplified Setup: ~$0.00 USD/day** (with no traffic)
- Container Apps: $0.00 when scaled to zero
- VNet: Free for basic usage
- No private endpoint or DNS zone costs

vs.

**Full Private Setup: ~$1.81 USD/day**
- Private Endpoint: $0.36/day
- Private DNS Zone: $1.20/day
- Container Apps: $0.00 when scaled to zero

## Internal DNS Pattern

Apps in the same Container Apps environment can reach each other using:
```
https://<app-name>.internal.<environment-name>.azurecontainerapps.io
```

This works because:
- All apps in the same environment share the same VNet
- Container Apps provides built-in service discovery
- No additional DNS configuration needed

## Next Steps

- **Deploy your own container**: Replace the hello-world image with your own application
- **Add more services**: Deploy additional container apps that communicate with each other
- **Implement CI/CD**: Set up automated deployments
- **Add monitoring**: Configure Application Insights for observability
- **Scale testing**: Experiment with different replica counts and auto-scaling rules

For more information, check out the [Azure Container Apps documentation](https://docs.microsoft.com/azure/container-apps/).
