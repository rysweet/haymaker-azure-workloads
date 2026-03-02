# Scenario: Linux VM Web Server

## Technology Area
Compute

## Scenario Description
Deploy a Linux VM running Ubuntu with Nginx web server to generate realistic compute telemetry.

## Goal
Create a functioning web server that serves HTTP traffic and generates Azure Monitor metrics.

## Azure Services Used
- Azure Virtual Machines
- Azure Virtual Network
- Network Security Group
- Public IP Address

## Phase 1: Deployment and Validation

```bash
# Set variables
RG_NAME="haymaker-${DEPLOYMENT_ID}-rg"
LOCATION="eastus"
VM_NAME="haymaker-vm-${DEPLOYMENT_ID}"

# Create resource group with tracking tag
az group create \
  --name $RG_NAME \
  --location $LOCATION \
  --tags HayMaker-managed=true deployment-id=$DEPLOYMENT_ID

# Create VM with Nginx
az vm create \
  --resource-group $RG_NAME \
  --name $VM_NAME \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard \
  --tags HayMaker-managed=true deployment-id=$DEPLOYMENT_ID

# Open port 80
az vm open-port --port 80 --resource-group $RG_NAME --name $VM_NAME

# Install Nginx
az vm run-command invoke \
  --resource-group $RG_NAME \
  --name $VM_NAME \
  --command-id RunShellScript \
  --scripts "sudo apt-get update && sudo apt-get install -y nginx && sudo systemctl start nginx"
```

## Phase 2: Operations and Management

```bash
# Generate traffic to the web server
VM_IP=$(az vm show -d -g $RG_NAME -n $VM_NAME --query publicIps -o tsv)
curl -s http://$VM_IP/ > /dev/null

# Check VM metrics
az monitor metrics list \
  --resource $(az vm show -g $RG_NAME -n $VM_NAME --query id -o tsv) \
  --metric "Percentage CPU" \
  --interval PT1M
```

## Phase 3: Cleanup

```bash
# Delete resource group (deletes all resources)
az group delete --name $RG_NAME --yes --no-wait
```
