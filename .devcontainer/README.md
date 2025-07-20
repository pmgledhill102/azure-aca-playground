# Azure Container Apps Jupyter Devcontainer

This devcontainer provides a complete environment for running Jupyter notebooks with Azure CLI and Container Apps support.

## Features

- **Python 3.13** with Jupyter Lab/Notebook support
- **Azure CLI** pre-installed and configured
- **Bash kernel** for running shell commands in notebooks
- **Container Apps extension** for Azure CLI
- **Docker-in-Docker** support for container operations
- **VS Code extensions** for Python, Jupyter, and Azure development

## Getting Started

1. **Open in VS Code**: Make sure you have the Dev Containers extension installed
2. **Reopen in Container**: When prompted, click "Reopen in Container" or use Command Palette > "Dev Containers: Reopen in Container"
3. **Wait for build**: The container will build and install all dependencies (first time may take a few minutes)
4. **Azure Login**: Once the container is ready, authenticate with Azure:

   ```bash
   az login
   ```

## Running the Notebooks

### Method 1: VS Code Jupyter Extension

1. Open either notebook file in VS Code
2. Select the Python kernel when prompted
3. Run cells individually or all at once

### Method 2: Jupyter Lab

1. Open a terminal in VS Code (Terminal > New Terminal)
2. Start Jupyter Lab:

   ```bash
   jupyter lab --ip=0.0.0.0 --port=8888 --no-browser --allow-root
   ```

3. Click on the forwarded port link or navigate to localhost:8888

## Available Notebooks

- **azure-container-apps-deployment.ipynb**: Public Container Apps deployment walkthrough
- **private-azure-container-apps-deployment.ipynb**: Private Container Apps deployment with VNet integration

## Azure CLI Extensions

The following Azure CLI extensions are pre-installed:

- `containerapp` - For Container Apps operations

## Persistent Azure Authentication

Your Azure CLI authentication will persist between container rebuilds thanks to the mounted `.azure` directory.

## Troubleshooting

### Container won't start

- Make sure Docker is running on your host machine
- Check that you have sufficient disk space for the container image

### Azure CLI authentication issues

- Run `az login` to re-authenticate
- Check your subscription context with `az account show`

### Jupyter kernel issues

- Restart the kernel: Kernel > Restart Kernel
- If bash kernel is missing, run: `python -m bash_kernel.install --user`

## Customization

To add more Python packages, edit `.devcontainer/requirements.txt` and rebuild the container.

To add system packages, edit `.devcontainer/Dockerfile` and rebuild the container.
