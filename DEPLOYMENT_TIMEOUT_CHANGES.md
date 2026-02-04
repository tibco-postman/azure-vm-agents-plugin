# Deployment Timeout Configuration Changes

## Summary

This change addresses the issue where VM deployments timeout after 10 minutes despite the `deploymentTimeout` being configured to 1200 seconds (20 minutes).

## Changes Made

### 1. Enhanced Logging (AzureVMCloud.java)

**Initial Deployment Log:**
```
INFO: Waiting for deployment deploy-20260204-123456 with VM ubuntu-0 to be completed. 
      Deployment timeout: 1200 seconds (20 minutes), max polling attempts: 240
```

**Progress Logs (every 60 seconds):**
```
INFO: Deployment deploy-20260204-123456 not yet finished (creating): virtualMachine:ubuntu-0 
      - waited 60 seconds (1 minutes), 1140 seconds remaining

INFO: Deployment deploy-20260204-123456 not yet finished (creating): virtualMachine:ubuntu-0 
      - waited 120 seconds (2 minutes), 1080 seconds remaining
```

**Purpose:**
- Confirms the actual timeout value being used
- Shows deployment progress  
- Helps diagnose if timeout is from this plugin or Azure SDK

### 2. UI Improvements (config.jelly)

**Before:** Deployment Timeout field in main configuration section
**After:** Deployment Timeout field in Advanced section

**Benefits:**
- Cleaner main configuration view
- Consistent with other advanced settings
- Still fully configurable via UI and JCasC

### 3. Better Documentation (help-deploymentTimeout.html)

Added clear information about:
- Default value: 1200 seconds (20 minutes)
- Minimum recommended: 600 seconds (10 minutes)
- What the timeout controls
- Note about potential Azure SDK HTTP timeouts

## How to Use

### Via UI:
1. Navigate to: Manage Jenkins → Nodes → Clouds → [Your Azure Cloud]
2. Click "Advanced" button
3. Set "Deployment Timeout" field (in seconds)
4. Default is 1200 if left blank

### Via JCasC:
```yaml
jenkins:
  clouds:
    - azureVM:
        name: "azure"
        azureCredentialsId: "azure-cred"
        deploymentTimeout: 1200  # 20 minutes
        # ... other settings
```

## Troubleshooting

### If deployments still timeout at 10 minutes:

1. **Check the logs** - Look for the initial deployment log:
   ```
   Waiting for deployment ... Deployment timeout: X seconds
   ```
   This confirms what timeout value is actually being used.

2. **Check progress logs** - Every minute you should see progress:
   ```
   Deployment ... waited X seconds (Y minutes), Z seconds remaining
   ```
   If these stop before reaching your configured timeout, it indicates an external timeout.

3. **Possible causes of 10-minute timeout despite correct configuration:**
   - **Azure SDK HTTP client timeout**: The Azure Java SDK may have a default 10-minute HTTP timeout
   - **Network/proxy timeout**: Network infrastructure between Jenkins and Azure
   - **Azure service timeout**: Azure ARM API may have internal timeouts

4. **Next steps if issue persists:**
   - Check Jenkins system logs for Azure SDK timeout messages
   - Verify network connectivity to Azure: `https://management.azure.com`
   - Consider increasing Azure SDK HTTP client timeout (requires code changes to HttpClientRetriever configuration)

## Backward Compatibility

- Existing configurations continue to work unchanged
- Default value remains 1200 seconds
- JCasC configurations with `deploymentTimeout` continue to work
- Field moved to Advanced section but still accessible

## Technical Details

### Code Flow:
1. User provisions a VM
2. `AzureVMCloud.createProvisionedAgent()` is called
3. Gets timeout from `getDeploymentTimeout()` (returns configured value or 1200 default)
4. Calculates max polling attempts: `maxTries = timeout / 5` (polls every 5 seconds)
5. Polls Azure deployment status up to maxTries times
6. Logs progress every 60 seconds
7. Returns agent when deployment succeeds or throws timeout exception

### Timeout Calculation:
- Sleep time: 5 seconds
- Timeout: 1200 seconds (configurable)
- Max tries: 1200 / 5 = 240 attempts
- Total max wait: 240 × 5 = 1200 seconds (20 minutes)
