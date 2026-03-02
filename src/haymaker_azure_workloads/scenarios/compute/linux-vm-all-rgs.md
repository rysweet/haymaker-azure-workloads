# Scenario: Linux VM in All Resource Groups

## Technology Area
Compute

## Scenario Description
Deploy a Linux VM with Nginx into every resource group within the current subscription. Use this for subscription-wide telemetry generation, security testing coverage, or ensuring every environment has baseline activity.

## Goal
Automatically deploy haymaker telemetry-generating VMs across an entire subscription for comprehensive coverage.

## Azure Services Used
- Azure Virtual Machines
- Network Security Group
- Public IP Address

## Required Configuration
- `RG_FILTER` - Optional regex pattern to filter resource groups (e.g., `wargaming-*`, `prod-*`)
- `EXCLUDE_PATTERN` - Optional pattern to exclude resource groups (e.g., `MC_*` for AKS managed RGs)
- `VM_PREFIX` - Prefix for VM names (default: `haymaker`)
- `MAX_DEPLOYMENTS` - Maximum number of VMs to deploy (default: unlimited)

## Phase 1: Deployment and Validation

```bash
# Set defaults
VM_PREFIX="${VM_PREFIX:-haymaker}"
EXCLUDE_PATTERN="${EXCLUDE_PATTERN:-^MC_|^DefaultResourceGroup|^NetworkWatcherRG|^LogAnalytics|^cloud-shell}"

# Get all resource groups
echo "Discovering resource groups..."
if [ -n "$RG_FILTER" ]; then
  RESOURCE_GROUPS=$(az group list --query "[?contains(name, '$RG_FILTER')].name" -o tsv)
else
  RESOURCE_GROUPS=$(az group list --query "[].name" -o tsv)
fi

# Filter out excluded patterns
FILTERED_RGS=$(echo "$RESOURCE_GROUPS" | grep -vE "$EXCLUDE_PATTERN" || true)

# Count and confirm
RG_COUNT=$(echo "$FILTERED_RGS" | grep -c . || echo 0)
echo "Found $RG_COUNT resource groups to deploy to"

if [ "$RG_COUNT" -eq 0 ]; then
  echo "No resource groups match the criteria. Exiting."
  exit 0
fi

# Apply max limit if set
if [ -n "$MAX_DEPLOYMENTS" ] && [ "$MAX_DEPLOYMENTS" -lt "$RG_COUNT" ]; then
  FILTERED_RGS=$(echo "$FILTERED_RGS" | head -n $MAX_DEPLOYMENTS)
  echo "Limited to $MAX_DEPLOYMENTS deployments"
fi

# Deploy to each resource group
DEPLOYED=0
FAILED=0

for RG in $FILTERED_RGS; do
  VM_NAME="${VM_PREFIX}-${DEPLOYMENT_ID}-$(echo $RG | md5sum | cut -c1-6)"
  
  echo "[$((DEPLOYED + FAILED + 1))/$RG_COUNT] Deploying $VM_NAME to $RG..."
  
  # Check if haymaker VM already exists in this RG
  EXISTING=$(az vm list -g $RG --query "[?starts_with(name, '${VM_PREFIX}')].name" -o tsv 2>/dev/null)
  if [ -n "$EXISTING" ]; then
    echo "  Skipping: Haymaker VM already exists ($EXISTING)"
    continue
  fi
  
  # Deploy VM (using --no-wait for parallel deployment)
  az vm create \
    --resource-group $RG \
    --name $VM_NAME \
    --image Ubuntu2204 \
    --size Standard_B1s \
    --admin-username azureuser \
    --generate-ssh-keys \
    --public-ip-sku Standard \
    --tags HayMaker-managed=true deployment-id=$DEPLOYMENT_ID scenario=linux-vm-all-rgs source-rg=$RG \
    --no-wait 2>/dev/null
  
  if [ $? -eq 0 ]; then
    DEPLOYED=$((DEPLOYED + 1))
    echo "  Started deployment"
  else
    FAILED=$((FAILED + 1))
    echo "  Failed to start deployment"
  fi
done

echo ""
echo "Deployment summary:"
echo "  Started: $DEPLOYED"
echo "  Failed: $FAILED"
echo "  Skipped: $((RG_COUNT - DEPLOYED - FAILED))"

# Wait for deployments to complete
echo ""
echo "Waiting for deployments to complete..."
sleep 60

# Install Nginx on all deployed VMs
echo "Installing Nginx on deployed VMs..."
for RG in $FILTERED_RGS; do
  VM_NAME="${VM_PREFIX}-${DEPLOYMENT_ID}-$(echo $RG | md5sum | cut -c1-6)"
  
  # Check if VM exists and is running
  VM_STATE=$(az vm show -g $RG -n $VM_NAME --query "provisioningState" -o tsv 2>/dev/null)
  if [ "$VM_STATE" = "Succeeded" ]; then
    az vm open-port --port 80 --resource-group $RG --name $VM_NAME --no-wait 2>/dev/null
    az vm run-command invoke \
      --resource-group $RG \
      --name $VM_NAME \
      --command-id RunShellScript \
      --scripts "sudo apt-get update && sudo apt-get install -y nginx && sudo systemctl start nginx" \
      --no-wait 2>/dev/null
    echo "  Configured: $VM_NAME"
  fi
done

echo "Phase 1 complete!"
```

## Phase 2: Operations and Management

```bash
# List all haymaker VMs across subscription
echo "=== Haymaker VMs in Subscription ==="
az vm list -d --query "[?tags.\"deployment-id\"=='$DEPLOYMENT_ID'].{Name:name, RG:resourceGroup, IP:publicIps, State:powerState}" -o table

# Generate traffic to all VMs
echo ""
echo "=== Generating Traffic ==="
for RG in $FILTERED_RGS; do
  VM_NAME="${VM_PREFIX}-${DEPLOYMENT_ID}-$(echo $RG | md5sum | cut -c1-6)"
  VM_IP=$(az vm show -d -g $RG -n $VM_NAME --query publicIps -o tsv 2>/dev/null)
  if [ -n "$VM_IP" ]; then
    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://$VM_IP/ 2>/dev/null || echo "000")
    echo "  $VM_NAME ($RG): $VM_IP - HTTP $HTTP_CODE"
  fi
done

# Summary metrics
TOTAL_VMS=$(az vm list --query "[?tags.\"deployment-id\"=='$DEPLOYMENT_ID'] | length(@)" -o tsv)
RUNNING_VMS=$(az vm list -d --query "[?tags.\"deployment-id\"=='$DEPLOYMENT_ID' && powerState=='VM running'] | length(@)" -o tsv)
echo ""
echo "Total VMs: $TOTAL_VMS"
echo "Running: $RUNNING_VMS"
```

## Phase 3: Cleanup

**IMPORTANT**: This cleanup phase ONLY deletes resources created by this specific haymaker deployment.
All other resources in each resource group are preserved untouched.

```bash
# ============================================================================
# SAFE CLEANUP: Only deletes resources tagged with this deployment ID
# - PRESERVES: All pre-existing resources in each resource group
# - PRESERVES: Resources created by other users, workloads, or deployments
# - PRESERVES: Resources without the deployment-id tag
# - DELETES: Only VMs and associated resources tagged with THIS deployment ID
# ============================================================================

echo "=== SAFE CLEANUP MODE ==="
echo "Deployment ID: $DEPLOYMENT_ID"
echo ""
echo "This will ONLY delete resources tagged with deployment-id=$DEPLOYMENT_ID"
echo "All other resources across all resource groups will be PRESERVED"
echo ""

# Find all resources tagged with this specific deployment ID
echo "Scanning subscription for resources from deployment $DEPLOYMENT_ID..."
VMS_TO_DELETE=$(az vm list --query "[?tags.\"deployment-id\"=='$DEPLOYMENT_ID'].{name:name, rg:resourceGroup}" -o tsv)

VM_COUNT=$(echo "$VMS_TO_DELETE" | grep -c . || echo 0)
echo "Found $VM_COUNT VMs tagged with deployment-id=$DEPLOYMENT_ID"

if [ "$VM_COUNT" -eq 0 ]; then
  echo "No VMs found for this deployment. Nothing to clean up."
  exit 0
fi

# List what will be deleted for transparency
echo ""
echo "The following VMs will be deleted (and ONLY these):"
echo "$VMS_TO_DELETE" | while read NAME RG; do
  if [ -n "$NAME" ]; then
    echo "  - $NAME (in $RG)"
  fi
done
echo ""

# Delete each VM - ONLY those with matching deployment-id tag
DELETED=0
SKIPPED=0

echo "$VMS_TO_DELETE" | while read NAME RG; do
  if [ -n "$NAME" ] && [ -n "$RG" ]; then
    # Double-check the deployment tag before deleting (defense in depth)
    ACTUAL_TAG=$(az vm show -g $RG -n $NAME --query "tags.\"deployment-id\"" -o tsv 2>/dev/null)
    
    if [ "$ACTUAL_TAG" = "$DEPLOYMENT_ID" ]; then
      echo "Deleting $NAME from $RG (verified: deployment-id=$DEPLOYMENT_ID)..."
      az vm delete --resource-group $RG --name $NAME --yes --no-wait 2>/dev/null
      DELETED=$((DELETED + 1))
    else
      echo "SKIPPING $NAME - tag mismatch (expected: $DEPLOYMENT_ID, found: $ACTUAL_TAG)"
      SKIPPED=$((SKIPPED + 1))
    fi
  fi
done

echo ""
echo "=== CLEANUP INITIATED ==="
echo "VM deletions running in background."
echo ""

# Clean up associated resources ONLY for this deployment
echo "Cleaning up associated resources tagged with deployment-id=$DEPLOYMENT_ID..."
TAGGED_RESOURCES=$(az resource list --tag deployment-id=$DEPLOYMENT_ID --query "[].{id:id, name:name, type:type}" -o tsv 2>/dev/null)

if [ -n "$TAGGED_RESOURCES" ]; then
  echo "Found additional tagged resources to clean up:"
  echo "$TAGGED_RESOURCES" | while read ID NAME TYPE; do
    if [ -n "$ID" ]; then
      echo "  Deleting: $NAME ($TYPE)"
      az resource delete --ids $ID --no-wait 2>/dev/null || true
    fi
  done
fi

echo ""
echo "=== CLEANUP COMPLETE ==="
echo "Only resources from deployment $DEPLOYMENT_ID were removed."
echo "All other resources in the subscription remain untouched."
```

## Usage Examples

```bash
# Deploy to ALL resource groups (be careful!)
haymaker deploy azure-infrastructure \
  --config scenario=linux-vm-all-rgs

# Deploy only to resource groups matching a pattern
haymaker deploy azure-infrastructure \
  --config scenario=linux-vm-all-rgs \
  --config rg_filter=wargaming

# Deploy with a maximum limit
haymaker deploy azure-infrastructure \
  --config scenario=linux-vm-all-rgs \
  --config max_deployments=10

# Deploy excluding certain patterns
haymaker deploy azure-infrastructure \
  --config scenario=linux-vm-all-rgs \
  --config exclude_pattern="^MC_|^test-"
```

## Safety Notes

⚠️ **This scenario can create many VMs quickly.** Use with caution:
- Always test with `--config max_deployments=1` first
- Use `--config rg_filter=` to limit scope
- Review excluded patterns to avoid managed resource groups
- Monitor costs - each VM costs ~$0.02/hour
