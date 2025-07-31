# Google Cloud EventArc and Cloud Logging Walkthrough

This interactive notebook guides you through creating two different Google Cloud event-driven architectures using Cloud Run services. You can run each cell individually or execute them in sequence to complete both deployment patterns.

## Overview

This notebook will help you:
- Install Google Cloud CLI prerequisites
- Authenticate with Google Cloud
- **Pattern 1: EventArc Direct Integration**
  - Create a Cloud Run service triggered by Compute Engine firewall rule changes via EventArc
  - Configure EventArc trigger for Compute Engine audit logs
  - Test by creating/deleting firewall rules in the GCP Console
- **Pattern 2: Cloud Logging Sink + Pub/Sub Pattern**
  - Create a Cloud Log Sink that sends logs to a Pub/Sub topic
  - Create a Cloud Run service triggered by Pub/Sub messages
  - Configure log filtering for specific events
  - Test by performing activities that generate the target log events
- Clean up all resources when done

**Note:** Make sure you have appropriate permissions in your Google Cloud project to create Cloud Run services, EventArc triggers, Pub/Sub topics, and manage IAM roles.

## Cost Considerations
- **Cloud Run**: Pay per request (free tier: 2M requests/month)
- **EventArc**: $0.60 per million events
- **Pub/Sub**: $40 per TiB of message throughput (free tier: 10GB/month)
- **Cloud Logging**: $0.50 per GB of logs ingested (free tier: 50GB/month)

**Estimated Cost**: $0-5/month for testing and light usage

## 1. Install Google Cloud CLI Prerequisites

First, check that the Google Cloud CLI (gcloud) is installed correctly with the required components.

If running in a devcontainer, you may need to install gcloud manually.


```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates gnupg curl
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
sudo apt-get update && sudo apt-get install -y google-cloud-cli

# Check Google Cloud CLI Prerequisites
echo "🔍 Checking Google Cloud CLI prerequisites..."
echo ""

# Check if gcloud CLI is installed
if command -v gcloud &> /dev/null; then
    echo "✅ Google Cloud CLI is installed: $(gcloud version --format='value(Google Cloud SDK)')"
else
    echo "❌ ERROR: Google Cloud CLI is not installed!"
    echo ""
    echo "📋 To install Google Cloud CLI:"
    echo "  • On macOS: brew install google-cloud-sdk"
    echo "  • On Ubuntu/Debian: curl https://sdk.cloud.google.com | bash"
    echo "  • On Windows: Download from https://cloud.google.com/sdk/docs/install"
    echo ""
    echo "🔧 Installing Google Cloud CLI now..."
    
    # Install gcloud CLI for Ubuntu/Debian (common in dev containers)
    curl https://sdk.cloud.google.com | bash
    exec -l $SHELL  # Reload shell
    
    if ! command -v gcloud &> /dev/null; then
        echo "❌ Installation failed. Please install manually."
        exit 1
    fi
fi

# Check if required components are installed
echo ""
echo "🔍 Checking required components..."

# List currently installed components
INSTALLED_COMPONENTS=$(gcloud components list --filter="State:Installed" --format="value(id)" 2>/dev/null | tr '\n' ' ')
echo "Installed components: $INSTALLED_COMPONENTS"

# Check for required components
REQUIRED_COMPONENTS=("gcloud" "gsutil")
MISSING_COMPONENTS=()

for component in "${REQUIRED_COMPONENTS[@]}"; do
    if [[ ! $INSTALLED_COMPONENTS =~ $component ]]; then
        MISSING_COMPONENTS+=("$component")
    fi
done

if [ ${#MISSING_COMPONENTS[@]} -eq 0 ]; then
    echo "✅ All required components are installed"
else
    echo "⚠️  Missing components: ${MISSING_COMPONENTS[*]}"
    echo "📋 To install missing components:"
    echo "  gcloud components install ${MISSING_COMPONENTS[*]}"
fi

echo ""
echo "🎯 Prerequisites check complete!"
```

    Hit:1 http://deb.debian.org/debian bullseye InRelease
    Hit:2 http://deb.debian.org/debian-security bullseye-security InRelease        
    Hit:1 http://deb.debian.org/debian bullseye InRelease
    Hit:2 http://deb.debian.org/debian-security bullseye-security InRelease        
    Hit:3 http://deb.debian.org/debian bullseye-updates InRelease                  
    Hit:3 http://deb.debian.org/debian bullseye-updates InRelease                  
    Hit:4 https://dl.yarnpkg.com/debian stable InRelease                           
    Hit:5 https://packages.microsoft.com/repos/azure-cli bullseye InRelease        
    Hit:4 https://dl.yarnpkg.com/debian stable InRelease                           
    Hit:5 https://packages.microsoft.com/repos/azure-cli bullseye InRelease        
    Hit:6 https://packages.microsoft.com/repos/microsoft-debian-bullseye-prod bullseye InRelease
    Hit:6 https://packages.microsoft.com/repos/microsoft-debian-bullseye-prod bullseye InRelease
    Reading package lists... 92%ing package lists... 0%Reading package lists... 0%Reading package lists... 0%Reading package lists... 0%Reading package lists... 0%Reading package lists... 0%Reading package lists... 92%Reading package lists... 92%Reading package lists... 92%Reading package lists... 97%Reading package lists... 97%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 99%Reading package lists... 99%Reading package lists... 99%Reading package lists... 99%Reading package lists... Done
    Reading package lists... 97%Reading package lists... 97%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 99%Reading package lists... 99%Reading package lists... 99%Reading package lists... 99%Reading package lists... Done
    Reading package lists... 97%eading package lists... 0%Reading package lists... 0%Reading package lists... 0%Reading package lists... 0%Reading package lists... 0%Reading package lists... 92%Reading package lists... 92%Reading package lists... 92%Reading package lists... 92%Reading package lists... 97%Reading package lists... 97%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 99%Reading package lists... 99%Reading package lists... 99%Reading package lists... 99%Reading package lists... 97%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 99%Reading package lists... 99%Reading package lists... 99%Reading package lists... 99%Reading package lists... Done
    Reading package lists... Done
    Building dependency tree... 0%Building dependency tree... 0%Building dependency tree... 0%Building dependency tree... 0%Building dependency tree... 50%Building dependency tree... 50%Building dependency tree... 50%Building dependency tree... 50%Building dependency tree... Done
    Reading state information... 0% Reading state information... 0%Reading state information... Done
    apt-transport-https is already the newest version (2.2.4).
    ca-certificates is already the newest version (20210119).
    gnupg is already the newest version (2.2.27-2+deb11u2).
    curl is already the newest version (7.74.0-1.3+deb11u15).
    Building dependency tree... Done
    Reading state information... 0% Reading state information... 0%Reading state information... Done
    apt-transport-https is already the newest version (2.2.4).
    ca-certificates is already the newest version (20210119).
    gnupg is already the newest version (2.2.27-2+deb11u2).
    curl is already the newest version (7.74.0-1.3+deb11u15).
    0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
    0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
    deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main
    deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
      0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
      0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
    100  1022  100  1022    0     0   4386      0 --:--:-- --:--:-- --:--:--  4386
    OK
    100  1022  100  1022    0     0   4386      0 --:--:-- --:--:-- --:--:--  4386
    OK
    Hit:1 http://deb.debian.org/debian bullseye InRelease
    Hit:2 http://deb.debian.org/debian-security bullseye-security InRelease        
    Hit:1 http://deb.debian.org/debian bullseye InRelease
    Hit:2 http://deb.debian.org/debian-security bullseye-security InRelease        
    Hit:3 http://deb.debian.org/debian bullseye-updates InRelease                  
    Hit:3 http://deb.debian.org/debian bullseye-updates InRelease                  
    Hit:4 https://packages.microsoft.com/repos/azure-cli bullseye InRelease        
    Hit:4 https://packages.microsoft.com/repos/azure-cli bullseye InRelease
    Hit:5 https://packages.microsoft.com/repos/microsoft-debian-bullseye-prod bullseye InRelease
    Hit:6 https://dl.yarnpkg.com/debian stable InRelease                           
    Hit:5 https://packages.microsoft.com/repos/microsoft-debian-bullseye-prod bullseye InRelease
    Hit:6 https://dl.yarnpkg.com/debian stable InRelease                           
    Get:7 https://packages.cloud.google.com/apt cloud-sdk InRelease [1618 B]       
    Get:7 https://packages.cloud.google.com/apt cloud-sdk InRelease [1618 B]
    Get:8 https://packages.cloud.google.com/apt cloud-sdk/main all Packages [1767 kB]
    Get:8 https://packages.cloud.google.com/apt cloud-sdk/main all Packages [1767 kB]
    Get:9 https://packages.cloud.google.com/apt cloud-sdk/main arm64 Packages [1579 kB]
    Get:9 https://packages.cloud.google.com/apt cloud-sdk/main arm64 Packages [1579 kB]
    Fetched 3347 kB in 1s (3101 kB/s)  
    Fetched 3347 kB in 1s (3101 kB/s)  
    Reading package lists... 0%Reading package lists... 0%Reading package lists... 0%Reading package lists... 0%Reading package lists... 0%Reading package lists... 0%Reading package lists... 86%Reading package lists... 86%Reading package lists... 86%Reading package lists... 86%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 94%Reading package lists... 94%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 94%Reading package lists... 94%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 99%Reading package lists... 99%Reading package lists... 99%Reading package lists... 99%Reading package lists... Done
    Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 99%Reading package lists... 99%Reading package lists... 99%Reading package lists... 99%Reading package lists... Done
    Reading package lists... 91%eading package lists... 0%Reading package lists... 0%Reading package lists... 0%Reading package lists... 0%Reading package lists... 0%Reading package lists... 86%Reading package lists... 86%Reading package lists... 86%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 91%Reading package lists... 94%Reading package lists... 94%Reading package lists... 94%Reading package lists... 94%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 99%Reading package lists... 99%Reading package lists... 99%Reading package lists... 99%Reading package lists... Done
    Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 98%Reading package lists... 99%Reading package lists... 99%Reading package lists... 99%Reading package lists... 99%Reading package lists... Done
    Building dependency tree... 0%Building dependency tree... 0%Building dependency tree... 0%Building dependency tree... 0%Building dependency tree... 50%Building dependency tree... 50%Building dependency tree... 50%Building dependency tree... 50%Building dependency tree... Done
    Reading state information... 0% Reading state information... 0%Reading state information... Done
    Building dependency tree... Done
    Reading state information... 0% Reading state information... 0%Reading state information... Done
    The following additional packages will be installed:
      google-cloud-cli-anthoscli
    Suggested packages:
      google-cloud-cli-app-engine-java google-cloud-cli-app-engine-python
      google-cloud-cli-pubsub-emulator google-cloud-cli-bigtable-emulator
      google-cloud-cli-datastore-emulator kubectl
    The following NEW packages will be installed:
      google-cloud-cli google-cloud-cli-anthoscli
    The following additional packages will be installed:
      google-cloud-cli-anthoscli
    Suggested packages:
      google-cloud-cli-app-engine-java google-cloud-cli-app-engine-python
      google-cloud-cli-pubsub-emulator google-cloud-cli-bigtable-emulator
      google-cloud-cli-datastore-emulator kubectl
    The following NEW packages will be installed:
      google-cloud-cli google-cloud-cli-anthoscli
    0 upgraded, 2 newly installed, 0 to remove and 0 not upgraded.
    Need to get 113 MB of archives.
    After this operation, 517 MB of additional disk space will be used.
    0 upgraded, 2 newly installed, 0 to remove and 0 not upgraded.
    Need to get 113 MB of archives.
    After this operation, 517 MB of additional disk space will be used.
    Get:1 https://packages.cloud.google.com/apt cloud-sdk/main arm64 google-cloud-cli arm64 530.0.0-0 [78.7 MB]
    Get:1 https://packages.cloud.google.com/apt cloud-sdk/main arm64 google-cloud-cli arm64 530.0.0-0 [78.7 MB]
    Get:2 https://packages.cloud.google.com/apt cloud-sdk/main arm64 google-cloud-cli-anthoscli arm64 530.0.0-0 [33.9 MB]
    Get:2 https://packages.cloud.google.com/apt cloud-sdk/main arm64 google-cloud-cli-anthoscli arm64 530.0.0-0 [33.9 MB]
    Fetched 113 MB in 11s (10.2 MB/s)                                              
    Fetched 113 MB in 11s (10.2 MB/s)                                              
    debconf: unable to initialize frontend: Dialog
    debconf: (Dialog frontend will not work on a dumb terminal, an emacs shell buffer, or without a controlling terminal.)
    debconf: falling back to frontend: Readline
    debconf: unable to initialize frontend: Dialog
    debconf: (Dialog frontend will not work on a dumb terminal, an emacs shell buffer, or without a controlling terminal.)
    debconf: falling back to frontend: Readline
    Selecting previously unselected package google-cloud-cli.
    Selecting previously unselected package google-cloud-cli.
    (Reading database ... 62788 files and directories currently installed.)
    (Reading database ... 62788 files and directories currently installed.)
    Preparing to unpack .../google-cloud-cli_530.0.0-0_arm64.deb ...
    Unpacking google-cloud-cli (530.0.0-0) ...
    Preparing to unpack .../google-cloud-cli_530.0.0-0_arm64.deb ...
    Unpacking google-cloud-cli (530.0.0-0) ...
    Selecting previously unselected package google-cloud-cli-anthoscli.
    Selecting previously unselected package google-cloud-cli-anthoscli.
    Preparing to unpack .../google-cloud-cli-anthoscli_530.0.0-0_arm64.deb ...
    Unpacking google-cloud-cli-anthoscli (530.0.0-0) ...
    Preparing to unpack .../google-cloud-cli-anthoscli_530.0.0-0_arm64.deb ...
    Unpacking google-cloud-cli-anthoscli (530.0.0-0) ...
    Setting up google-cloud-cli (530.0.0-0) ...
    Setting up google-cloud-cli (530.0.0-0) ...
    Setting up google-cloud-cli-anthoscli (530.0.0-0) ...
    Processing triggers for man-db (2.9.4-2) ...
    Setting up google-cloud-cli-anthoscli (530.0.0-0) ...
    Processing triggers for man-db (2.9.4-2) ...
    🔍 Checking Google Cloud CLI prerequisites...
    🔍 Checking Google Cloud CLI prerequisites...
    
    
    ERROR: (gcloud.version) Expected ) in projection expression [flattened[no-pad,separator=" "] value(Google *HERE* Cloud SDK)].
    ERROR: (gcloud.version) Expected ) in projection expression [flattened[no-pad,separator=" "] value(Google *HERE* Cloud SDK)].
    ✅ Google Cloud CLI is installed: 
    ✅ Google Cloud CLI is installed: 
    
    
    🔍 Checking required components...
    🔍 Checking required components...
    Installed components: app-engine-go package-go-module cbt bigtable cloud-datastore-emulator cloud-firestore-emulator pubsub-emulator cloud-run-proxy cloud-sql-proxy docker-credential-gcr kustomize log-streaming managed-flink-client minikube local-extract skaffold spanner-cli terraform-tools anthos-auth config-connector enterprise-certificate-proxy app-engine-java app-engine-python app-engine-python-extras gke-gcloud-auth-plugin kpt kubectl kubectl-oidc pkg bq gsutil core gcloud-crc32c alpha beta istioctl 
    Installed components: app-engine-go package-go-module cbt bigtable cloud-datastore-emulator cloud-firestore-emulator pubsub-emulator cloud-run-proxy cloud-sql-proxy docker-credential-gcr kustomize log-streaming managed-flink-client minikube local-extract skaffold spanner-cli terraform-tools anthos-auth config-connector enterprise-certificate-proxy app-engine-java app-engine-python app-engine-python-extras gke-gcloud-auth-plugin kpt kubectl kubectl-oidc pkg bq gsutil core gcloud-crc32c alpha beta istioctl 
    ✅ All required components are installed
    ✅ All required components are installed
    
    
    🎯 Prerequisites check complete!
    🎯 Prerequisites check complete!


## 2. Configure Google Cloud Authentication

Login to Google Cloud and verify your project context.


```bash
# Check if already logged in
echo "🔍 Checking Google Cloud authentication status..."

if ! gcloud auth list --filter=status:ACTIVE --format="value(account)" | head -n1 > /dev/null 2>&1; then
    echo "❌ Not logged in to Google Cloud"
    echo "🔐 Please authenticate with Google Cloud..."
    echo ""
    
    # Attempt login
    gcloud auth login --no-browser
    
    # Verify login was successful
    if ! gcloud auth list --filter=status:ACTIVE --format="value(account)" | head -n1 > /dev/null 2>&1; then
        echo "❌ Authentication failed. Please try again manually:"
        echo "   gcloud auth login --no-browser"
        exit 1
    fi
    
    echo "✅ Successfully authenticated!"
else
    echo "✅ Already logged in to Google Cloud"
    ACTIVE_ACCOUNT=$(gcloud auth list --filter=status:ACTIVE --format="value(account)" | head -n1)
    echo "   Active account: $ACTIVE_ACCOUNT"
fi

echo ""
echo "📋 Available projects:"
gcloud projects list --format="table(projectId,name,projectNumber)" --limit=10

echo ""
# Check if a project is already set
CURRENT_PROJECT=$(gcloud config get-value project 2>/dev/null)

if [ -z "$CURRENT_PROJECT" ] || [ "$CURRENT_PROJECT" = "(unset)" ]; then
    echo "⚠️  No project is currently set!"
    echo ""
    echo "Please set your project using one of these methods:"
    echo "  1. Run: gcloud config set project YOUR_PROJECT_ID"
    echo "  2. Export: export GOOGLE_CLOUD_PROJECT=YOUR_PROJECT_ID"
    echo ""
    echo "� Choose a project ID from the list above and run:"
    echo "   gcloud config set project PROJECT_ID"
    echo ""
    echo "⏸️  Execution paused. Set your project and re-run this cell."
    exit 1
else
    echo "✅ Current project: $CURRENT_PROJECT"
    
    # Verify the project exists and we have access
    if gcloud projects describe $CURRENT_PROJECT > /dev/null 2>&1; then
        PROJECT_NAME=$(gcloud projects describe $CURRENT_PROJECT --format="value(name)")
        PROJECT_NUMBER=$(gcloud projects describe $CURRENT_PROJECT --format="value(projectNumber)")
        echo "   Project name: $PROJECT_NAME"
        echo "   Project number: $PROJECT_NUMBER"
        
        # Set the project in environment variable for consistency
        export GOOGLE_CLOUD_PROJECT=$CURRENT_PROJECT
        echo "   Environment variable set: GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT"
    else
        echo "❌ Cannot access project: $CURRENT_PROJECT"
        echo "   Please check the project ID and your permissions"
        exit 1
    fi
fi

echo ""
echo "🎯 Authentication check complete!"
```

    🔍 Checking Google Cloud authentication status...
    ✅ Already logged in to Google Cloud
    ✅ Already logged in to Google Cloud
       Active account: Paul@pmgledhill.com
       Active account: Paul@pmgledhill.com
    
    
    📋 Available projects:
    📋 Available projects:
    PROJECT_ID    NAME         PROJECT_NUMBER
    dfc-gaming    dfc-gaming   116284661734
    neukin-barn   neukin-barn  71695094478
    play-pen-pup  play-pen     728241575576
    temp-lbg      temp-lbg     465690885103
    PROJECT_ID    NAME         PROJECT_NUMBER
    dfc-gaming    dfc-gaming   116284661734
    neukin-barn   neukin-barn  71695094478
    play-pen-pup  play-pen     728241575576
    temp-lbg      temp-lbg     465690885103
    
    
    ✅ Current project: play-pen-pup
    ✅ Current project: play-pen-pup
       Project name: play-pen
       Project number: 728241575576
       Environment variable set: GOOGLE_CLOUD_PROJECT=play-pen-pup
       Project name: play-pen
       Project number: 728241575576
       Environment variable set: GOOGLE_CLOUD_PROJECT=play-pen-pup
    
    
    🎯 Authentication check complete!
    🎯 Authentication check complete!


## 3. Set Deployment Variables

Define the configuration variables for your event-driven deployment. You can modify these values as needed for your specific setup.


```bash
# Get the current project ID with multiple fallback methods
PROJECT_ID_FROM_CONFIG=$(gcloud config get-value project 2>/dev/null)
PROJECT_ID_FROM_ENV=${GOOGLE_CLOUD_PROJECT:-}

# Use config value first, then environment variable
if [ -n "$PROJECT_ID_FROM_CONFIG" ] && [ "$PROJECT_ID_FROM_CONFIG" != "(unset)" ]; then
    export PROJECT_ID="$PROJECT_ID_FROM_CONFIG"
elif [ -n "$PROJECT_ID_FROM_ENV" ]; then
    export PROJECT_ID="$PROJECT_ID_FROM_ENV"
    # Also set gcloud config to match
    gcloud config set project "$PROJECT_ID"
else
    echo "❌ ERROR: No project ID found!"
    echo "Please set your project using:"
    echo "  gcloud config set project YOUR_PROJECT_ID"
    echo "Or run the previous cell to authenticate and select a project."
    exit 1
fi

# Verify we can access the project
if ! gcloud projects describe "$PROJECT_ID" > /dev/null 2>&1; then
    echo "❌ ERROR: Cannot access project: $PROJECT_ID"
    echo "Please check the project ID and your permissions."
    exit 1
fi

echo "🔧 Setting up deployment variables..."
echo "✅ Using project: $PROJECT_ID"

# Set other deployment variables
export REGION="us-central1"
export ZONE="us-central1-a"

# Pattern 1: EventArc Direct Integration
export EVENTARC_SERVICE_NAME="firewall-monitor-service"
export EVENTARC_TRIGGER_NAME="firewall-changes-trigger"
export EVENTARC_SA="eventarc-service-account"

# Pattern 2: Pub/Sub + Cloud Logging
export PUBSUB_SERVICE_NAME="log-processor-service"
export PUBSUB_TOPIC_NAME="firewall-logs-topic"
export PUBSUB_SUBSCRIPTION_NAME="firewall-logs-subscription"
export LOG_SINK_NAME="firewall-logs-sink"
export PUBSUB_SA="pubsub-service-account"

# Test resources
export TEST_FIREWALL_RULE="test-rule-for-events"

# Display the variables
echo ""
echo "📋 Deployment Configuration:"
echo "=========================="
echo "Project ID:              $PROJECT_ID"
echo "Region:                  $REGION"
echo "Zone:                    $ZONE"
echo ""
echo "=== EventArc Pattern ==="
echo "Service Name:            $EVENTARC_SERVICE_NAME"
echo "Trigger Name:            $EVENTARC_TRIGGER_NAME"
echo "Service Account:         $EVENTARC_SA"
echo ""
echo "=== Pub/Sub Pattern ==="
echo "Service Name:            $PUBSUB_SERVICE_NAME"
echo "Topic Name:              $PUBSUB_TOPIC_NAME"
echo "Subscription Name:       $PUBSUB_SUBSCRIPTION_NAME"
echo "Log Sink Name:           $LOG_SINK_NAME"
echo "Service Account:         $PUBSUB_SA"
echo ""
echo "Test Firewall Rule:      $TEST_FIREWALL_RULE"

# Verify we can access the project APIs
echo ""
echo "🔍 Verifying project access..."
PROJECT_INFO=$(gcloud projects describe "$PROJECT_ID" --format="value(name,projectNumber)" 2>/dev/null)
if [ -n "$PROJECT_INFO" ]; then
    echo "✅ Project access confirmed"
    echo "✅ Ready to proceed with deployment!"
else
    echo "❌ Unable to verify project access"
    exit 1
fi
```

    🔧 Setting up deployment variables...
    ✅ Using project: play-pen-pup
    ✅ Using project: play-pen-pup
    
    
    📋 Deployment Configuration:
    📋 Deployment Configuration:
    ==========================
    ==========================
    Project ID:              play-pen-pup
    Project ID:              play-pen-pup
    Region:                  us-central1
    Region:                  us-central1
    Zone:                    us-central1-a
    Zone:                    us-central1-a
    
    
    === EventArc Pattern ===
    === EventArc Pattern ===
    Service Name:            firewall-monitor-service
    Service Name:            firewall-monitor-service
    Trigger Name:            firewall-changes-trigger
    Trigger Name:            firewall-changes-trigger
    Service Account:         eventarc-service-account
    Service Account:         eventarc-service-account
    
    === Pub/Sub Pattern ===
    
    === Pub/Sub Pattern ===
    Service Name:            log-processor-service
    Service Name:            log-processor-service
    Topic Name:              firewall-logs-topic
    Topic Name:              firewall-logs-topic
    Subscription Name:       firewall-logs-subscription
    Subscription Name:       firewall-logs-subscription
    Log Sink Name:           firewall-logs-sink
    Log Sink Name:           firewall-logs-sink
    Service Account:         pubsub-service-account
    Service Account:         pubsub-service-account
    
    
    Test Firewall Rule:      test-rule-for-events
    Test Firewall Rule:      test-rule-for-events
    
    
    🔍 Verifying project access...
    🔍 Verifying project access...
    ✅ Project access confirmed
    ✅ Ready to proceed with deployment!
    ✅ Project access confirmed
    ✅ Ready to proceed with deployment!


## 4. Enable Required Google Cloud APIs

Enable all the APIs needed for EventArc, Cloud Run, Pub/Sub, and Cloud Logging.


```bash
# Enable required APIs
echo "🔄 Enabling required Google Cloud APIs..."

REQUIRED_APIS=(
    "run.googleapis.com"              # Cloud Run
    "eventarc.googleapis.com"         # EventArc
    "pubsub.googleapis.com"           # Pub/Sub
    "logging.googleapis.com"          # Cloud Logging
    "compute.googleapis.com"          # Compute Engine (for firewall rules)
    "cloudbuild.googleapis.com"       # Cloud Build (for container builds)
    "artifactregistry.googleapis.com" # Artifact Registry
)

for api in "${REQUIRED_APIS[@]}"; do
    echo "  • Enabling $api..."
    gcloud services enable $api
done

echo ""
echo "⏳ Waiting for APIs to be fully enabled..."
sleep 30

echo "✅ All required APIs are now enabled!"
```

    🔄 Enabling required Google Cloud APIs...
      • Enabling run.googleapis.com...
      • Enabling run.googleapis.com...
      • Enabling eventarc.googleapis.com...
      • Enabling eventarc.googleapis.com...
      • Enabling pubsub.googleapis.com...
      • Enabling pubsub.googleapis.com...
      • Enabling logging.googleapis.com...
      • Enabling logging.googleapis.com...
      • Enabling compute.googleapis.com...
      • Enabling compute.googleapis.com...
      • Enabling cloudbuild.googleapis.com...
      • Enabling cloudbuild.googleapis.com...
      • Enabling artifactregistry.googleapis.com...
      • Enabling artifactregistry.googleapis.com...
    
    
    ⏳ Waiting for APIs to be fully enabled...
    ⏳ Waiting for APIs to be fully enabled...
    ✅ All required APIs are now enabled!
    ✅ All required APIs are now enabled!


## 5. Create Service Accounts and IAM Roles

Create dedicated service accounts for both patterns with appropriate permissions.


```bash
# Create service account for EventArc pattern
echo "🔧 Creating EventArc service account..."

gcloud iam service-accounts create $EVENTARC_SA \
    --display-name="EventArc Cloud Run Service Account" \
    --description="Service account for EventArc-triggered Cloud Run service"

# Grant necessary roles to EventArc service account
EVENTARC_SA_EMAIL="${EVENTARC_SA}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${EVENTARC_SA_EMAIL}" \
    --role="roles/eventarc.eventReceiver"

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${EVENTARC_SA_EMAIL}" \
    --role="roles/run.invoker"

echo "✅ EventArc service account created: $EVENTARC_SA_EMAIL"
```


```bash
# Create service account for Pub/Sub pattern
echo "🔧 Creating Pub/Sub service account..."

gcloud iam service-accounts create $PUBSUB_SA \
    --display-name="Pub/Sub Cloud Run Service Account" \
    --description="Service account for Pub/Sub-triggered Cloud Run service"

# Grant necessary roles to Pub/Sub service account
PUBSUB_SA_EMAIL="${PUBSUB_SA}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${PUBSUB_SA_EMAIL}" \
    --role="roles/run.invoker"

echo "✅ Pub/Sub service account created: $PUBSUB_SA_EMAIL"
```

# Pattern 1: EventArc Direct Integration

## 6. Create Container Images for Cloud Run Services

First, let's create simple Cloud Run services that will respond to events.


```bash
# Create directory for EventArc service source code
mkdir -p ./eventarc-service
cd ./eventarc-service

# Create a simple Python Flask app for EventArc
cat > app.py << 'EOF'
import os
import json
from flask import Flask, request, jsonify
import logging

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)

@app.route("/", methods=["POST"])
def handle_event():
    """Handle EventArc CloudEvent"""
    try:
        # Get the CloudEvent data
        event_data = request.get_json()
        
        # Log the received event
        app.logger.info(f"Received EventArc event: {json.dumps(event_data, indent=2)}")
        
        # Extract relevant information
        event_type = request.headers.get('ce-type', 'unknown')
        event_source = request.headers.get('ce-source', 'unknown')
        event_subject = request.headers.get('ce-subject', 'unknown')
        
        response = {
            "message": "Event processed successfully",
            "event_type": event_type,
            "event_source": event_source,
            "event_subject": event_subject,
            "data": event_data
        }
        
        app.logger.info(f"Processing event: {event_type} from {event_source}")
        
        return jsonify(response), 200
        
    except Exception as e:
        app.logger.error(f"Error processing event: {str(e)}")
        return jsonify({"error": str(e)}), 500

@app.route("/health", methods=["GET"])
def health_check():
    """Health check endpoint"""
    return jsonify({"status": "healthy"}), 200

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8080))
    app.run(host="0.0.0.0", port=port, debug=False)
EOF

# Create requirements.txt
cat > requirements.txt << 'EOF'
Flask==2.3.3
gunicorn==21.2.0
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8080

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "1", "app:app"]
EOF

echo "✅ EventArc service source code created"
cd ..
```


```bash
# Build and push EventArc service container image
echo "🔨 Building EventArc service container..."

# Create Artifact Registry repository if it doesn't exist
gcloud artifacts repositories create cloud-run-services \
    --repository-format=docker \
    --location=$REGION \
    --description="Repository for Cloud Run services" \
    --quiet || true

# Configure Docker authentication for Artifact Registry
gcloud auth configure-docker ${REGION}-docker.pkg.dev --quiet

# Build and push the EventArc service image
EVENTARC_IMAGE="${REGION}-docker.pkg.dev/${PROJECT_ID}/cloud-run-services/${EVENTARC_SERVICE_NAME}:latest"

cd ./eventarc-service
gcloud builds submit --tag $EVENTARC_IMAGE .
cd ..

echo "✅ EventArc service image built: $EVENTARC_IMAGE"
```

## 7. Deploy EventArc Cloud Run Service

Deploy the Cloud Run service that will be triggered by EventArc.


```bash
# Deploy EventArc Cloud Run service
echo "🚀 Deploying EventArc Cloud Run service..."

gcloud run deploy $EVENTARC_SERVICE_NAME \
    --image=$EVENTARC_IMAGE \
    --region=$REGION \
    --service-account=$EVENTARC_SA_EMAIL \
    --no-allow-unauthenticated \
    --memory=512Mi \
    --cpu=1 \
    --min-instances=0 \
    --max-instances=10 \
    --timeout=60s \
    --platform=managed

# Get the service URL
EVENTARC_SERVICE_URL=$(gcloud run services describe $EVENTARC_SERVICE_NAME \
    --region=$REGION \
    --format="value(status.url)")

echo "✅ EventArc Cloud Run service deployed: $EVENTARC_SERVICE_URL"
```

## 8. Create EventArc Trigger

Create an EventArc trigger that will monitor Compute Engine firewall rule changes and trigger the Cloud Run service.


```bash
# Create EventArc trigger for Compute Engine firewall changes
echo "🔗 Creating EventArc trigger for firewall changes..."

gcloud eventarc triggers create $EVENTARC_TRIGGER_NAME \
    --location=$REGION \
    --destination-run-service=$EVENTARC_SERVICE_NAME \
    --destination-run-region=$REGION \
    --event-filters="type=google.cloud.audit.log.v1.written" \
    --event-filters="serviceName=compute.googleapis.com" \
    --event-filters="methodName=v1.compute.firewalls.insert" \
    --event-filters-path-pattern="resourceName=/projects/${PROJECT_ID}/global/firewalls/*" \
    --service-account=$EVENTARC_SA_EMAIL

echo "✅ EventArc trigger created: $EVENTARC_TRIGGER_NAME"
echo "📡 The trigger will monitor firewall rule creation events"
```


```bash
# Create additional EventArc trigger for firewall deletions
EVENTARC_DELETE_TRIGGER_NAME="${EVENTARC_TRIGGER_NAME}-delete"

echo "🔗 Creating EventArc trigger for firewall deletions..."

gcloud eventarc triggers create $EVENTARC_DELETE_TRIGGER_NAME \
    --location=$REGION \
    --destination-run-service=$EVENTARC_SERVICE_NAME \
    --destination-run-region=$REGION \
    --event-filters="type=google.cloud.audit.log.v1.written" \
    --event-filters="serviceName=compute.googleapis.com" \
    --event-filters="methodName=v1.compute.firewalls.delete" \
    --event-filters-path-pattern="resourceName=/projects/${PROJECT_ID}/global/firewalls/*" \
    --service-account=$EVENTARC_SA_EMAIL

echo "✅ Additional EventArc trigger created: $EVENTARC_DELETE_TRIGGER_NAME"
echo "📡 This trigger will monitor firewall rule deletion events"
```

# Pattern 2: Cloud Logging Sink + Pub/Sub

## 9. Create Pub/Sub Topic and Subscription

Set up the Pub/Sub infrastructure for the logging pattern.


```bash
# Create Pub/Sub topic
echo "📨 Creating Pub/Sub topic..."

gcloud pubsub topics create $PUBSUB_TOPIC_NAME

# Create Pub/Sub subscription
gcloud pubsub subscriptions create $PUBSUB_SUBSCRIPTION_NAME \
    --topic=$PUBSUB_TOPIC_NAME \
    --ack-deadline=60 \
    --message-retention-duration=7d

echo "✅ Pub/Sub topic and subscription created:"
echo "  Topic: $PUBSUB_TOPIC_NAME"
echo "  Subscription: $PUBSUB_SUBSCRIPTION_NAME"
```

## 10. Create Pub/Sub Cloud Run Service

Create a Cloud Run service that processes messages from Pub/Sub.


```bash
# Create directory for Pub/Sub service source code
mkdir -p ./pubsub-service
cd ./pubsub-service

# Create a Python Flask app for Pub/Sub
cat > app.py << 'EOF'
import os
import json
import base64
from flask import Flask, request, jsonify
import logging

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)

@app.route("/", methods=["POST"])
def handle_pubsub():
    """Handle Pub/Sub push message"""
    try:
        # Get the Pub/Sub message
        envelope = request.get_json()
        
        if not envelope:
            app.logger.error("No Pub/Sub message received")
            return jsonify({"error": "Bad Request: no Pub/Sub message"}), 400

        pubsub_message = envelope.get('message')
        if not pubsub_message:
            app.logger.error("No message field in Pub/Sub envelope")
            return jsonify({"error": "Bad Request: invalid Pub/Sub message format"}), 400

        # Decode the message data
        message_data = None
        if pubsub_message.get('data'):
            message_data = base64.b64decode(pubsub_message['data']).decode('utf-8')
            try:
                message_data = json.loads(message_data)
            except json.JSONDecodeError:
                pass  # Keep as string if not JSON

        # Get message attributes
        attributes = pubsub_message.get('attributes', {})
        message_id = pubsub_message.get('messageId')
        
        # Log the received message
        app.logger.info(f"Received Pub/Sub message ID: {message_id}")
        app.logger.info(f"Message data: {json.dumps(message_data, indent=2) if isinstance(message_data, dict) else message_data}")
        app.logger.info(f"Message attributes: {json.dumps(attributes, indent=2)}")
        
        # Process the log entry if it's a Cloud Logging message
        if isinstance(message_data, dict) and 'logName' in message_data:
            log_name = message_data.get('logName', 'unknown')
            severity = message_data.get('severity', 'INFO')
            resource = message_data.get('resource', {})
            
            app.logger.info(f"Processing Cloud Log: {log_name} (severity: {severity})")
            
            # Extract specific information for firewall-related logs
            if 'compute.googleapis.com' in log_name:
                proto_payload = message_data.get('protoPayload', {})
                method_name = proto_payload.get('methodName', 'unknown')
                resource_name = proto_payload.get('resourceName', 'unknown')
                
                app.logger.info(f"Compute Engine event: {method_name} on {resource_name}")
        
        response = {
            "message": "Pub/Sub message processed successfully",
            "messageId": message_id,
            "attributes": attributes,
            "data_preview": str(message_data)[:200] + "..." if len(str(message_data)) > 200 else str(message_data)
        }
        
        return jsonify(response), 200
        
    except Exception as e:
        app.logger.error(f"Error processing Pub/Sub message: {str(e)}")
        return jsonify({"error": str(e)}), 500

@app.route("/health", methods=["GET"])
def health_check():
    """Health check endpoint"""
    return jsonify({"status": "healthy"}), 200

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8080))
    app.run(host="0.0.0.0", port=port, debug=False)
EOF

# Create requirements.txt
cat > requirements.txt << 'EOF'
Flask==2.3.3
gunicorn==21.2.0
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8080

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "1", "app:app"]
EOF

echo "✅ Pub/Sub service source code created"
cd ..
```


```bash
# Build and push Pub/Sub service container image
echo "🔨 Building Pub/Sub service container..."

PUBSUB_IMAGE="${REGION}-docker.pkg.dev/${PROJECT_ID}/cloud-run-services/${PUBSUB_SERVICE_NAME}:latest"

cd ./pubsub-service
gcloud builds submit --tag $PUBSUB_IMAGE .
cd ..

echo "✅ Pub/Sub service image built: $PUBSUB_IMAGE"
```

## 11. Deploy Pub/Sub Cloud Run Service

Deploy the Cloud Run service that will be triggered by Pub/Sub messages.


```bash
# Deploy Pub/Sub Cloud Run service
echo "🚀 Deploying Pub/Sub Cloud Run service..."

gcloud run deploy $PUBSUB_SERVICE_NAME \
    --image=$PUBSUB_IMAGE \
    --region=$REGION \
    --service-account=$PUBSUB_SA_EMAIL \
    --no-allow-unauthenticated \
    --memory=512Mi \
    --cpu=1 \
    --min-instances=0 \
    --max-instances=10 \
    --timeout=60s \
    --platform=managed

# Get the service URL
PUBSUB_SERVICE_URL=$(gcloud run services describe $PUBSUB_SERVICE_NAME \
    --region=$REGION \
    --format="value(status.url)")

echo "✅ Pub/Sub Cloud Run service deployed: $PUBSUB_SERVICE_URL"
```

## 12. Create Pub/Sub Push Subscription

Configure Pub/Sub to push messages directly to the Cloud Run service.


```bash
# Delete the existing pull subscription and create a push subscription
gcloud pubsub subscriptions delete $PUBSUB_SUBSCRIPTION_NAME --quiet

# Create push subscription
gcloud pubsub subscriptions create $PUBSUB_SUBSCRIPTION_NAME \
    --topic=$PUBSUB_TOPIC_NAME \
    --push-endpoint=$PUBSUB_SERVICE_URL \
    --ack-deadline=60 \
    --message-retention-duration=7d

# Grant Pub/Sub the ability to create tokens for the service
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:service-$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')@gcp-sa-pubsub.iam.gserviceaccount.com" \
    --role="roles/iam.serviceAccountTokenCreator"

echo "✅ Push subscription configured to trigger Cloud Run service"
```

## 13. Create Cloud Logging Sink

Create a log sink that sends specific log entries to the Pub/Sub topic.


```bash
# Create Cloud Logging sink for firewall-related events
echo "📊 Creating Cloud Logging sink..."

# Log filter for Compute Engine firewall operations
LOG_FILTER='protoPayload.serviceName="compute.googleapis.com"
AND protoPayload.methodName:("firewalls.insert" OR "firewalls.delete" OR "firewalls.patch")
AND severity>=INFO'

gcloud logging sinks create $LOG_SINK_NAME \
    pubsub.googleapis.com/projects/$PROJECT_ID/topics/$PUBSUB_TOPIC_NAME \
    --log-filter="$LOG_FILTER" \
    --description="Sink for Compute Engine firewall events"

# Get the sink's service account
SINK_SA=$(gcloud logging sinks describe $LOG_SINK_NAME --format="value(writerIdentity)")

# Grant the sink's service account permission to publish to Pub/Sub
gcloud pubsub topics add-iam-policy-binding $PUBSUB_TOPIC_NAME \
    --member="$SINK_SA" \
    --role="roles/pubsub.publisher"

echo "✅ Cloud Logging sink created: $LOG_SINK_NAME"
echo "📨 Sink service account: $SINK_SA"
echo "🔍 Log filter: $LOG_FILTER"
```

## 14. Test Both Patterns

Now let's test both event-driven patterns by creating and deleting a test firewall rule.


```bash
# Create a test firewall rule to trigger events
echo "🧪 Creating test firewall rule to trigger events..."

gcloud compute firewall-rules create $TEST_FIREWALL_RULE \
    --description="Test rule for EventArc and logging demonstrations" \
    --direction=INGRESS \
    --action=ALLOW \
    --source-ranges=10.0.0.0/8 \
    --rules=tcp:8080 \
    --target-tags=test-target \
    --quiet

echo "✅ Test firewall rule created: $TEST_FIREWALL_RULE"
echo ""
echo "🔄 This should trigger both:"
echo "  1. EventArc trigger → Cloud Run service"
echo "  2. Cloud Logging sink → Pub/Sub → Cloud Run service"
echo ""
echo "⏳ Wait a few minutes for events to propagate..."
```


```bash
# Wait and then delete the test firewall rule
echo "⏳ Waiting 30 seconds before deleting the test rule..."
sleep 30

echo "🗑️  Deleting test firewall rule to trigger deletion events..."

gcloud compute firewall-rules delete $TEST_FIREWALL_RULE --quiet

echo "✅ Test firewall rule deleted: $TEST_FIREWALL_RULE"
echo ""
echo "🔄 This should trigger both deletion events:"
echo "  1. EventArc deletion trigger → Cloud Run service"
echo "  2. Cloud Logging sink → Pub/Sub → Cloud Run service"
```

## 15. Check Service Logs

Examine the logs from both Cloud Run services to verify they received and processed the events.


```bash
# Check EventArc service logs
echo "📊 EventArc Cloud Run Service Logs (last 50 entries):"
echo "=================================================="

gcloud logging read "resource.type=cloud_run_revision
AND resource.labels.service_name=$EVENTARC_SERVICE_NAME
AND severity>=INFO" \
    --limit=50 \
    --format="table(timestamp,severity,textPayload)" \
    --freshness=10m

echo ""
echo "📊 Pub/Sub Cloud Run Service Logs (last 50 entries):"
echo "=================================================="

gcloud logging read "resource.type=cloud_run_revision
AND resource.labels.service_name=$PUBSUB_SERVICE_NAME  
AND severity>=INFO" \
    --limit=50 \
    --format="table(timestamp,severity,textPayload)" \
    --freshness=10m
```

## 16. Manual Testing via GCP Console

You can manually test the event-driven systems by performing the following actions in the Google Cloud Console:

### Testing EventArc Pattern:
1. Go to **VPC Network > Firewall** in the GCP Console
2. Click **CREATE FIREWALL RULE**
3. Create a simple rule with:
   - Name: `manual-test-rule`
   - Direction: Ingress
   - Action: Allow
   - Targets: All instances in the network
   - Source IP ranges: `10.0.0.0/8`
   - Protocols: TCP, Port: 8080
4. Click **CREATE**
5. After a few minutes, delete the rule

### Testing Pub/Sub Pattern:
The same firewall rule operations will also trigger the Cloud Logging sink pattern.

### Checking Results:
- View Cloud Run service logs in the **Cloud Run** section of the GCP Console
- Check EventArc triggers in the **EventArc** section  
- Monitor Pub/Sub messages in the **Pub/Sub** section

## 17. Cleanup Resources

When you're done with your event-driven setup, use the following commands to clean up the resources to avoid ongoing charges.

**⚠️ Warning:** These commands will permanently delete your resources. Make sure you no longer need them before proceeding.


```bash
# Delete EventArc triggers
echo "🗑️  Deleting EventArc triggers..."
gcloud eventarc triggers delete $EVENTARC_TRIGGER_NAME --location=$REGION --quiet
gcloud eventarc triggers delete "${EVENTARC_TRIGGER_NAME}-delete" --location=$REGION --quiet

# Delete Cloud Run services
echo "🗑️  Deleting Cloud Run services..."
gcloud run services delete $EVENTARC_SERVICE_NAME --region=$REGION --quiet
gcloud run services delete $PUBSUB_SERVICE_NAME --region=$REGION --quiet

# Delete Pub/Sub resources
echo "🗑️  Deleting Pub/Sub resources..."
gcloud pubsub subscriptions delete $PUBSUB_SUBSCRIPTION_NAME --quiet
gcloud pubsub topics delete $PUBSUB_TOPIC_NAME --quiet

# Delete Cloud Logging sink
echo "🗑️  Deleting Cloud Logging sink..."
gcloud logging sinks delete $LOG_SINK_NAME --quiet

# Delete service accounts
echo "🗑️  Deleting service accounts..."
gcloud iam service-accounts delete $EVENTARC_SA_EMAIL --quiet
gcloud iam service-accounts delete $PUBSUB_SA_EMAIL --quiet

# Delete container images (optional)
echo "🗑️  Deleting container images..."
gcloud artifacts repositories delete cloud-run-services \
    --location=$REGION --quiet || true

echo "✅ Cleanup completed!"
```


```bash
# Clean up local files and directories
echo "🧹 Cleaning up local files..."

rm -rf ./eventarc-service
rm -rf ./pubsub-service

echo "✅ Local cleanup completed!"
```

## Summary

🎉 **Congratulations!** You have successfully implemented two different event-driven patterns in Google Cloud:

### Pattern 1: EventArc Direct Integration
1. ✅ Created a Cloud Run service to handle EventArc events
2. ✅ Set up EventArc triggers for Compute Engine firewall operations  
3. ✅ Configured direct event routing from Google Cloud services to Cloud Run
4. ✅ Tested with firewall rule creation and deletion

### Pattern 2: Cloud Logging Sink + Pub/Sub
1. ✅ Created a Cloud Run service to process Pub/Sub messages
2. ✅ Set up a Pub/Sub topic and push subscription
3. ✅ Configured a Cloud Logging sink to route specific log entries
4. ✅ Implemented log filtering and message processing
5. ✅ Tested the complete pipeline from audit logs to service execution

## Key Differences Between Patterns

| Aspect | EventArc Direct | Cloud Logging + Pub/Sub |
|--------|----------------|-------------------------|
| **Latency** | Lower (direct routing) | Higher (multiple hops) |
| **Filtering** | Limited filtering options | Advanced log filtering |
| **Reliability** | High (managed by Google) | High (with Pub/Sub durability) |
| **Scalability** | Auto-scaling Cloud Run | Pub/Sub + Auto-scaling |
| **Cost** | $0.60 per million events | $0.50/GB logs + $40/TiB Pub/Sub |
| **Complexity** | Simple setup | More components to manage |
| **Event Types** | CloudEvents from supported services | Any Cloud Logging entry |
| **Processing** | Real-time | Near real-time with queuing |

## Cost Summary (Monthly Estimates)

**Light Testing Usage:**
- **EventArc**: ~$0.01 (1,000 events)
- **Pub/Sub Pattern**: ~$0.50 (basic log ingestion)
- **Cloud Run**: $0.00 (within free tier)
- **Total**: ~$0.51/month

## Use Cases

### Choose EventArc Direct When:
- You need low latency event processing
- Working with supported Google Cloud services
- Simple event routing requirements
- Real-time processing is critical

### Choose Cloud Logging + Pub/Sub When:
- You need complex log filtering
- Working with custom applications generating logs
- Need message durability and replay capability
- Implementing fan-out patterns to multiple consumers
- Batch processing requirements

## Next Steps

- **Add More Event Types**: Extend EventArc triggers to other Google Cloud services
- **Implement Dead Letter Queues**: Add error handling for failed message processing
- **Add Monitoring**: Set up alerting for failed events and service errors
- **Scale Testing**: Implement load testing for high-volume scenarios
- **Security Hardening**: Add VPC connectors and additional security layers
- **Multi-Region**: Deploy across multiple regions for high availability

For more advanced Google Cloud event-driven patterns, check out the [EventArc documentation](https://cloud.google.com/eventarc/docs) and [Pub/Sub documentation](https://cloud.google.com/pubsub/docs).
