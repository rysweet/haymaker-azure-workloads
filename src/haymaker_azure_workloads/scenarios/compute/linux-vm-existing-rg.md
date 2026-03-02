# Scenario: Linux VM in Existing Resource Group

## Technology Area
Compute

## Scenario Description
Deploy a Linux VM with Nginx into an existing resource group. Use this when you want to add haymaker telemetry generation to existing infrastructure without creating new resource groups.

## Goal
Add a telemetry-generating VM to an existing resource group for testing, CTF scenarios, or augmenting existing environments.

## Azure Services Used
- Azure Virtual Machines
- Network Security Group
- Public IP Address

## Required Configuration
- `TARGET_RG` - The existing resource group name (required)
- `VM_NAME` - Custom VM name (optional, defaults to `haymaker-vm-${DEPLOYMENT_ID}`)
- `LOCATION` - Azure region (optional, inherits from resource group)

## Phase 1: Deployment and Validation

```bash
# Validate required config
if [ -z "$TARGET_RG" ]; then
  echo "ERROR: TARGET_RG is required. Set with --config target_rg=<resource-group-name>"
  exit 1
fi

# Verify resource group exists
RG_EXISTS=$(az group exists --name $TARGET_RG)
if [ "$RG_EXISTS" != "true" ]; then
  echo "ERROR: Resource group '$TARGET_RG' does not exist"
  exit 1
fi

# Set variables
VM_NAME="${VM_NAME:-haymaker-vm-${DEPLOYMENT_ID}}"
LOCATION=$(az group show --name $TARGET_RG --query location -o tsv)

echo "Deploying $VM_NAME to existing resource group: $TARGET_RG ($LOCATION)"

# Create VM in existing resource group
az vm create \
  --resource-group $TARGET_RG \
  --name $VM_NAME \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard \
  --tags HayMaker-managed=true deployment-id=$DEPLOYMENT_ID scenario=linux-vm-existing-rg

# Open port 80
az vm open-port --port 80 --resource-group $TARGET_RG --name $VM_NAME

# Install Nginx
az vm run-command invoke \
  --resource-group $TARGET_RG \
  --name $VM_NAME \
  --command-id RunShellScript \
  --scripts "sudo apt-get update && sudo apt-get install -y nginx && sudo systemctl start nginx"

# Get and display public IP
VM_IP=$(az vm show -d -g $TARGET_RG -n $VM_NAME --query publicIps -o tsv)
echo "VM deployed successfully!"
echo "Public IP: $VM_IP"
echo "Test with: curl http://$VM_IP/"
```

## Phase 2: Operations and Management

```bash
# Generate traffic to the web server
VM_IP=$(az vm show -d -g $TARGET_RG -n $VM_NAME --query publicIps -o tsv)
curl -s http://$VM_IP/ > /dev/null

# Check VM metrics
az monitor metrics list \
  --resource $(az vm show -g $TARGET_RG -n $VM_NAME --query id -o tsv) \
  --metric "Percentage CPU" \
  --interval PT1M

# List all haymaker VMs in this resource group
az vm list -g $TARGET_RG --query "[?tags.\"HayMaker-managed\"=='true'].{Name:name, State:provisioningState}" -o table
```

## Phase 3: Cleanup

**IMPORTANT**: This cleanup phase ONLY deletes resources created by this haymaker deployment. 
All other resources in the resource group are preserved untouched.

```bash
# ============================================================================
# SAFE CLEANUP: Only deletes resources tagged with this deployment ID
# - PRESERVES: All pre-existing resources in the resource group
# - PRESERVES: Resources created by other users or workloads
# - DELETES: Only the VM and associated resources created by THIS deployment
# ============================================================================

echo "=== SAFE CLEANUP MODE ==="
echo "Deployment ID: $DEPLOYMENT_ID"
echo "Target RG: $TARGET_RG"
echo ""
echo "This will ONLY delete resources tagged with deployment-id=$DEPLOYMENT_ID"
echo "All other resources in $TARGET_RG will be PRESERVED"
echo ""

# Verify the VM belongs to this deployment before deleting
VM_DEPLOYMENT_TAG=$(az vm show -g $TARGET_RG -n $VM_NAME --query "tags.\"deployment-id\"" -o tsv 2>/dev/null)

if [ "$VM_DEPLOYMENT_TAG" != "$DEPLOYMENT_ID" ]; then
  echo "ERROR: VM $VM_NAME does not belong to deployment $DEPLOYMENT_ID"
  echo "Expected tag: $DEPLOYMENT_ID, Found: $VM_DEPLOYMENT_TAG"
  echo "Aborting cleanup to prevent accidental deletion"
  exit 1
fi

# Delete only the haymaker VM we created
echo "Deleting VM: $VM_NAME (verified: deployment-id=$DEPLOYMENT_ID)"
az vm delete --resource-group $TARGET_RG --name $VM_NAME --yes

# Clean up ONLY the associated resources created with this VM
# These follow Azure's default naming convention for resources created with az vm create
echo "Cleaning up associated resources..."
az network nic delete --resource-group $TARGET_RG --name ${VM_NAME}VMNic 2>/dev/null || true
az network public-ip delete --resource-group $TARGET_RG --name ${VM_NAME}PublicIP 2>/dev/null || true
az network nsg delete --resource-group $TARGET_RG --name ${VM_NAME}NSG 2>/dev/null || true

# Delete OS disk by querying for exact name with deployment tag
DISK_NAME=$(az disk list -g $TARGET_RG --query "[?tags.\"deployment-id\"=='$DEPLOYMENT_ID'].name" -o tsv 2>/dev/null | head -1)
if [ -n "$DISK_NAME" ]; then
  az disk delete --resource-group $TARGET_RG --name $DISK_NAME --yes 2>/dev/null || true
fi

echo ""
echo "=== CLEANUP COMPLETE ==="
echo "Resource group '$TARGET_RG' preserved with all other resources intact."
echo "Only haymaker deployment $DEPLOYMENT_ID resources were removed."
```

## Usage Example

```bash
# Deploy to specific existing resource group
haymaker deploy azure-infrastructure \
  --config scenario=linux-vm-existing-rg \
  --config target_rg=my-existing-resource-group \
  --config vm_name=haymaker-telemetry-vm

# Deploy to CTF resource group
haymaker deploy azure-infrastructure \
  --config scenario=linux-vm-existing-rg \
  --config target_rg=wargaming-m003-v1-base
```
