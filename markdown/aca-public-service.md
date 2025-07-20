# Public Azure Container Apps Deployment Playbook

This interactive notebook guides you through deploying a public Azure Container Apps service using Azure CLI commands. You can run each cell individually or execute them in sequence to complete the full public deployment.

## Overview

This notebook will help you:
- Install Azure CLI prerequisites
- Authenticate with Azure
- Create a Container Apps Environment
- Deploy a public Container App with external ingress
- Retrieve the application URL
- Test the deployment
- Clean up resources when done

**Note:** Make sure you have appropriate permissions in your Azure subscription to create Container Apps resources.

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

Define the configuration variables for your public Container App deployment. You can modify these values as needed for your specific deployment.


```bash
# Set variables for public setup with unique resource group name
export LOCATION="uksouth"
export RESOURCE_GROUP="public-container-apps-rg"
export CONTAINER_APP_ENV="public-container-app-env"
export CONTAINER_APP_NAME="public-hello-world-app"

# Display the variables
echo "Resource Group:     $RESOURCE_GROUP"
echo "Location:           $LOCATION"
echo "Container App Env:  $CONTAINER_APP_ENV"
echo "Container App Name: $CONTAINER_APP_NAME"
```

## 4. Register Resource Providers


```bash
# Register resource providers
echo "🔄 Registering resource providers..."

az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights

echo "⏳ Waiting for resource providers to be registered..."

# Wait for Microsoft.App to be registered
while [ "$(az provider show --namespace Microsoft.App --query registrationState -o tsv)" != "Registered" ]; do
    echo "  • Microsoft.App: $(az provider show --namespace Microsoft.App --query registrationState -o tsv)"
    sleep 10
done

# Wait for Microsoft.OperationalInsights to be registered  
while [ "$(az provider show --namespace Microsoft.OperationalInsights --query registrationState -o tsv)" != "Registered" ]; do
    echo "  • Microsoft.OperationalInsights: $(az provider show --namespace Microsoft.OperationalInsights --query registrationState -o tsv)"
    sleep 10
done

echo "✅ All resource providers are now registered!"
echo "  • Microsoft.App: Registered"
echo "  • Microsoft.OperationalInsights: Registered"
```

## 5. Create Azure Resource Group

Create the resource group that will contain all your public Container App resources.


```bash
# Create resource group if it doesn't exist
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

## 6. Create Container Apps Environment

Create the Container Apps environment that will host your container applications.


```bash
# Create the Container Apps Environment
az containerapp env create \
  --name $CONTAINER_APP_ENV \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION
```

## 7. Deploy Public Container App

Deploy the hello-world container application with external ingress for public access.


```bash
# Create the Container App
az containerapp create \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINER_APP_ENV \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --target-port 80 \
  --ingress external \
  --min-replicas 0 \
  --max-replicas 5 \
  --cpu 0.25 \
  --memory 0.5Gi
```

## 8. Retrieve Container App URL

Get the publicly accessible URL of your deployed container application.


```bash
# Get the Container App URL
CONTAINER_APP_URL=$(az containerapp show \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query properties.configuration.ingress.fqdn \
  --output tsv)

echo "Container App URL: https://$CONTAINER_APP_URL"
```

## 9. Test Public Access

Test connectivity to your publicly deployed Container App.


```bash
# Test the Container App URL
echo "🔍 Testing Container App connectivity..."
echo "Container App URL: https://$CONTAINER_APP_URL"
echo ""

# Test with curl
if command -v curl &> /dev/null; then
    echo "📡 Testing with curl..."
    curl -s -o /dev/null -w "Response code: %{http_code}\nTotal time: %{time_total}s\n" "https://$CONTAINER_APP_URL" || echo "❌ Curl test failed"
    echo ""
fi

# Also test with wget as fallback
if command -v wget &> /dev/null; then
    echo "📡 Testing with wget..."
    wget -qO- --timeout=10 "https://$CONTAINER_APP_URL" | head -c 200 || echo "❌ Wget test failed"
    echo ""
    echo ""
fi

echo "✅ You can also open the URL in your browser: https://$CONTAINER_APP_URL"
```

## 10. Verification Commands

Run these commands to verify your deployment status and configuration.


```bash
# Check Container Apps Environment status
az containerapp env show \
  --name $CONTAINER_APP_ENV \
  --resource-group $RESOURCE_GROUP \
  --query "{name:name, location:location, provisioningState:properties.provisioningState}" \
  --output table
```


```bash
# Check Container App status
az containerapp show \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "{name:name, provisioningState:properties.provisioningState, fqdn:properties.configuration.ingress.fqdn, ingressType:properties.configuration.ingress.external}" \
  --output table
```


```bash
# Pause for user to play before tidying resources
exit 1
```

## 11. Cleanup Resources

When you're done with your public Container App setup, use the following commands to clean up the resources to avoid ongoing charges.

**⚠️ Warning:** These commands will permanently delete your resources. Make sure you no longer need them before proceeding.


```bash
# Delete the entire resource group (this will delete all resources)
# ⚠️ This will delete ALL resources in the resource group!
az group delete \
  --name $RESOURCE_GROUP \
  --yes --no-wait
```

## Summary

🎉 **Congratulations!** You have successfully:

1. ✅ Installed Azure CLI prerequisites
2. ✅ Authenticated with Azure
3. ✅ Set deployment variables
4. ✅ Registered required resource providers
5. ✅ Created a resource group
6. ✅ Created a Container Apps environment
7. ✅ Deployed a public containerized application with external ingress
8. ✅ Retrieved the application URL
9. ✅ Tested public connectivity
10. ✅ Verified deployment status
11. ✅ (Optional) Cleaned up resources

## Key Differences from Private Setup

- **Public Ingress**: Container App uses `--ingress external` for public internet access
- **No VNet Integration**: Uses Azure's managed networking without custom VNet setup
- **Direct Internet Access**: Application is directly accessible from the internet
- **Simplified Architecture**: No private endpoints, DNS zones, or jumper containers needed
- **Scale-to-Zero Architecture**: App can scale to zero when not in use, eliminating compute costs
- **Simple Testing**: Direct HTTP access without requiring internal network connectivity

## Cost Considerations

**Daily Cost Estimate: ~$0.00 USD** (with no traffic)

Key cost components:
- **Container Apps**: $0.00 when scaled to zero (pay only when processing requests)
- **Container Apps Environment**: Shared compute resources included in request billing
- **No Additional Networking**: No private endpoints or DNS zones required

## Next Steps

- **Deploy your own container**: Replace the hello-world image with your own containerized application
- **Configure custom domains**: Set up custom domain names for your Container Apps
- **Implement CI/CD**: Set up automated deployments using GitHub Actions or Azure DevOps
- **Add monitoring**: Configure Application Insights and Log Analytics for monitoring
- **Scale testing**: Experiment with different replica counts and resource allocations
- **Security hardening**: Add authentication, rate limiting, and other security measures
- **Private deployment**: Try the private Container Apps deployment for internal applications

For more advanced Container Apps features and private networking scenarios, check out the [Azure Container Apps documentation](https://docs.microsoft.com/azure/container-apps/).
