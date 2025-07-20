# Azure Container Apps Playground

This repository contains interactive playbooks and documentation for working with Azure Container Apps.

## Repository Structure

```
├── playbooks/           # Interactive Jupyter notebooks (.ipynb files)
├── markdown/           # Auto-generated markdown documentation
├── .github/workflows/  # GitHub Actions for automation
└── .devcontainer/     # Development container configuration
```

## Playbooks

Interactive Jupyter notebooks that guide you through Azure Container Apps deployments:

- **`playbooks/private-azure-container-apps-deployment.ipynb`** - Complete guide for deploying private Container Apps with VNet integration
- **`playbooks/azure-container-apps-deployment.ipynb`** - Basic Container Apps deployment guide

## Documentation

The `markdown/` folder contains auto-generated markdown versions of the playbooks:

- **`markdown/private-azure-container-apps-deployment.md`** - Static documentation for private deployments
- **`markdown/azure-container-apps-deployment.md`** - Static documentation for basic deployments

## Development

### Prerequisites

This repository includes a dev container with all necessary tools pre-installed:
- Azure CLI
- Python with Jupyter
- Node.js and npm
- Docker CLI

### Automatic Documentation Generation

When you update any notebook in the `playbooks/` folder, a GitHub Action automatically:
1. Removed all 'Output' from the Playbooks
2. Converts the notebook to markdown format
3. Saves the markdown file in the `markdown/` folder
4. Commits the updated documentation

This ensures the markdown documentation is always in sync with the interactive playbooks.

### Manual Conversion

To manually convert a notebook to markdown:

```bash
# Convert a specific notebook
jupyter nbconvert --to markdown playbooks/your-notebook.ipynb --output-dir markdown

# Convert all notebooks
find playbooks -name "*.ipynb" -exec jupyter nbconvert --to markdown {} --output-dir markdown \;
```

## Getting Started

1. Open this repository in VS Code with the Dev Container extension
2. Navigate to the `playbooks/` folder
3. Open any `.ipynb` file to start the interactive experience
4. Follow the step-by-step instructions in the notebooks

## Cost Considerations

The playbooks include cost estimates and cleanup procedures to help you manage Azure spending. Always review the cost sections and clean up resources when done with testing.

## Private Service Architecture Details

### Azure Container Apps Environment

The private deployment creates a **Consumption Plan** Azure Container Apps Environment with the following characteristics:

- **Plan Type**: Consumption (Pay-per-request)
- **Workload Profile**: Default consumption profile
- **Pricing Model**: $0.000016/vCPU-second + $0.000002/GiB-second
- **Scaling**: Automatic scale-to-zero capabilities
- **Networking**: VNet-integrated with internal-only ingress

### Network Architecture

```
Azure Subscription
└── Resource Group: private-container-apps-rg
    ├── Virtual Network: private-vnet (10.0.0.0/16)
    │   ├── Container Apps Subnet: container-apps-subnet (10.0.0.0/21)
    │   │   └── Delegated to: Microsoft.App/environments
    │   └── Private Endpoint Subnet: private-endpoint-subnet (10.0.8.0/24)
    │       └── Private Endpoint IP: Dynamic allocation
    │
    ├── Private DNS Zone: privatelink.azurecontainerapps.io
    │   ├── DNS Link: container-apps-dns-link → private-vnet
    │   └── A Record: private-hello-world-app → Private Endpoint IP
    │
    └── Private Endpoint: container-app-private-endpoint
        ├── Connection: container-app-connection
        └── Target: Container Apps Environment (managedEnvironments)
```

### Application Stack

```
Container Apps Environment: private-container-app-env
├── Configuration
│   ├── Location: UK South
│   ├── Internal Only: true
│   ├── VNet Integration: private-vnet/container-apps-subnet
│   └── Infrastructure Subnet: /21 delegation
│
├── Private Container App: private-hello-world-app
│   ├── Image: mcr.microsoft.com/azuredocs/containerapps-helloworld:latest
│   ├── Ingress: internal (HTTPS only)
│   ├── Target Port: 80
│   ├── Internal FQDN: private-hello-world-app.internal.[environment-id]
│   ├── Resource Allocation
│   │   ├── CPU: 0.25 cores
│   │   ├── Memory: 0.5 GiB
│   │   ├── Min Replicas: 0 (scale-to-zero)
│   │   └── Max Replicas: 5
│   └── Network Access: VNet internal only
│
└── Jumper Container App: jumper-container-app
    ├── Image: mcr.microsoft.com/azure-cli:latest
    ├── Purpose: Testing and connectivity validation
    ├── Timeout: 600 seconds (10 minutes)
    ├── Environment Variables
    │   └── APP_URL: https://[private-app-internal-fqdn]
    ├── Resource Allocation
    │   ├── CPU: 0.25 cores
    │   ├── Memory: 0.5 GiB
    │   ├── Min Replicas: 0 (scale-to-zero)
    │   └── Max Replicas: 1
    └── Network Access: Same VNet as private app
```

### Cost Breakdown (Private Deployment)

**Daily Estimate: ~$1.81 USD** (with no active traffic)

| Component | Daily Cost | Monthly Cost | Notes |
|-----------|------------|--------------|-------|
| Private Endpoint | $0.36 | $10.80 | Always-on networking component |
| Private DNS Zone | $1.20 | $15.00 | DNS resolution service |
| Container Apps (Idle) | $0.00 | $0.00 | Scale-to-zero when no requests |
| Jumper App (Idle) | $0.00 | $0.00 | Only costs during testing |
| VNet & Subnets | $0.00 | $0.00 | No additional charges |
| **Total** | **$1.56** | **$25.80** | Base infrastructure cost |

*Additional consumption charges apply during active usage:*
- **vCPU**: $0.000016/second per core
- **Memory**: $0.000002/second per GiB
- **HTTP Requests**: Included in consumption billing

### Public vs Private Comparison

| Feature | Public Deployment | Private Deployment |
|---------|-------------------|-------------------|
| **Ingress Type** | External (Internet) | Internal (VNet only) |
| **Network Setup** | Azure-managed | Custom VNet + Private Endpoints |
| **DNS Resolution** | Public DNS | Private DNS Zone |
| **Testing Method** | Direct HTTP access | Jumper container required |
| **Base Cost** | $0.00/day | $1.56/day |
| **Security** | Internet-exposed | Private network only |
| **Complexity** | Simple | Advanced networking |

