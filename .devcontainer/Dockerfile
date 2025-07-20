FROM mcr.microsoft.com/devcontainers/python:3.13-bullseye

# Install additional system packages
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    git \
    vim \
    jq \
    openssl \
    && rm -rf /var/lib/apt/lists/*

# Install Jupyter and required packages
COPY requirements.txt /tmp/requirements.txt
RUN pip install --no-cache-dir -r /tmp/requirements.txt

# Install bash kernel for Jupyter (system-wide)
RUN python -m bash_kernel.install

# Create a non-root user workspace
USER vscode
WORKDIR /workspaces

# Set environment variables for Azure CLI
ENV AZURE_CONFIG_DIR=/home/vscode/.azure
ENV PYTHONPATH=/workspaces

# Expose Jupyter port
EXPOSE 8888
