# Azure OS Provisioning Timeout Fix

## Problem

VMs were failing after 600 seconds (10 minutes) with error:
```
Deployment Failed: Microsoft.Compute/virtualMachines:toutb9fc62 - Conflict - 
OS Provisioning for VM 'toutb9fc62' did not finish in the allotted time.
The VM may still finish provisioning successfully. Please check provisioning state later.
```

**Timeline:**
- Time 0-405s: VM provisioning in progress (state: Running)
- Time 600s (10 min): Azure OS provisioning timeout → Deployment failed
- Jenkins timeout (1200s): Never reached

## Root Cause

### Two Different Timeouts

1. **Azure OS Provisioning Timeout: 600 seconds (10 minutes)**
   - Hardcoded by Azure platform
   - Cannot be configured by users
   - Applies to OS-level provisioning inside the VM
   - Triggers "Conflict" error with specific message

2. **Jenkins Deployment Timeout: Configurable (default 1200s)**
   - Set in Jenkins Cloud configuration (Advanced section)
   - Controls how long Jenkins waits for deployment to complete
   - Default: 1200 seconds (20 minutes)

### The Issue

Azure's 600-second OS provisioning timeout was triggering first, causing immediate failure even though:
- Jenkins was configured to wait 1200 seconds
- Azure's own error message says: "**The VM may still finish provisioning successfully**"
- The VM might complete if given more time

## Solution

### Code Changes

The fix detects Azure OS provisioning timeout errors and **continues waiting** instead of failing immediately:

```java
// Detect Azure OS provisioning timeout
boolean isOsProvisioningTimeout = finalStatusMessage != null 
        && finalStatusMessage.contains("OS Provisioning") 
        && finalStatusMessage.contains("did not finish in the allotted time");

if (isOsProvisioningTimeout) {
    // Log warning but continue waiting
    LOGGER.log(Level.WARNING,
            "Azure OS provisioning timeout detected after {0} seconds. "
            + "This is an Azure-side timeout (~10 minutes), not Jenkins timeout. "
            + "Azure reports the VM may still finish provisioning. "
            + "Continuing to wait up to Jenkins timeout ({1} seconds total).",
            waitedSeconds, timeoutInSeconds);
    // Don't throw - let VM potentially complete
} else {
    // Other failures still fail immediately
    throw AzureCloudException.create(...);
}
```

### Behavior

**Before Fix:**
```
0-600s:   VM provisioning (creating/running)
600s:     Azure OS timeout → IMMEDIATE FAILURE ❌
1200s:    Never reached
```

**After Fix:**
```
0-600s:   VM provisioning (creating/running)
600s:     Azure OS timeout → Log WARNING, continue waiting ⚠️
600-1200s: VM might still complete ✓
1200s:    Jenkins timeout → Fail only if VM didn't complete
```

### Expected Logs

When Azure OS provisioning timeout occurs at 600 seconds:

```
INFO: Waiting for deployment deploy-123 with VM ubuntu-0 to be completed. 
      Deployment timeout: 1200 seconds (20 minutes), max polling attempts: 240

INFO: Deployment deploy-123 not yet finished (Running)...waited 60s, 1140s remaining
INFO: Deployment deploy-123 not yet finished (Running)...waited 120s, 1080s remaining
...
INFO: Deployment deploy-123 not yet finished (Running)...waited 600s, 600s remaining

WARNING: Azure OS provisioning timeout detected for VM ubuntu-0 after 600 seconds. 
         This is an Azure-side timeout (typically ~10 minutes), not Jenkins timeout. 
         Azure reports the VM may still finish provisioning. 
         Continuing to wait up to Jenkins timeout (1200 seconds total). 
         Error: Conflict - OS Provisioning for VM 'ubuntu-0' did not finish in the allotted time...

INFO: Deployment deploy-123 not yet finished (Running)...waited 660s, 540s remaining
INFO: Deployment deploy-123 not yet finished (Running)...waited 720s, 480s remaining
...

# One of two outcomes:
# 1. VM completes successfully:
INFO: VM available: ubuntu-0

# 2. VM still doesn't complete by Jenkins timeout:
ERROR: Deployment deploy-123 failed: Jenkins deployment timeout reached (1200 seconds). 
       The VM deployment did not complete within the configured timeout. 
       This may be due to slow VM provisioning or Azure OS provisioning issues. 
       Consider: 1) Increasing deploymentTimeout in Advanced settings, 
       2) Using a properly prepared/generalized image, 
       3) Checking Azure service health.
```

## When This Helps

This fix is beneficial when:

1. **Image provisioning is slow but succeeds eventually**
   - VM takes 11-20 minutes to provision
   - Would complete if given full 1200s instead of failing at 600s

2. **Azure platform experiencing delays**
   - Azure services temporarily slow
   - VM would succeed with a bit more time

3. **Custom images with complex setup**
   - Image has extensive initialization scripts
   - Takes longer than 10 minutes to fully provision

## When This Doesn't Help

This fix won't solve:

1. **Improperly prepared images**
   - Image not generalized correctly
   - VM will never complete, even with more time
   - Solution: Fix image preparation (see Azure docs)

2. **Very large Jenkins timeout still not enough**
   - Even 1200s+ is insufficient
   - Solution: Fix underlying image/provisioning issues

3. **Azure platform issues**
   - Azure services down or degraded
   - Solution: Wait for Azure service recovery

## Recommendations

### For Users Experiencing This Issue

1. **Verify your Jenkins timeout**:
   - Cloud Configuration → Advanced → Deployment Timeout
   - Should be at least 1200 seconds (20 minutes)
   - Can increase to 1800-2400s if needed

2. **Check image preparation**:
   - Ensure VM image is properly generalized
   - Windows: https://learn.microsoft.com/azure/virtual-machines/windows/prepare-for-upload-vhd-image
   - Linux: https://learn.microsoft.com/azure/virtual-machines/linux/create-upload-generic

3. **Monitor logs**:
   - Look for "Azure OS provisioning timeout detected" warning
   - Check if VM eventually completes after warning
   - If VM never completes, image likely has issues

4. **Consider shared image gallery**:
   - For >20 concurrent VMs, use Azure Shared Image Gallery
   - See: https://aka.ms/movetosig

### For Plugin Developers

This fix:
- ✅ Handles specific error pattern for OS provisioning timeout
- ✅ Continues waiting as Azure suggests ("may still finish")
- ✅ Respects Jenkins' longer timeout configuration
- ✅ Provides clear logging to distinguish timeout types
- ✅ Backward compatible - other failures still fail immediately

## Technical Details

### Azure OS Provisioning Timeout

Azure has an internal timeout (~10 minutes / 600 seconds) for OS-level provisioning. This is:
- **Separate from deployment creation** (which is fast)
- **Separate from Jenkins polling timeout** (configurable, default 1200s)
- **Not configurable** by users
- **May resolve itself** if given more time (per Azure's error message)

### Error Detection

The fix detects this specific error by checking if the error message contains:
1. "OS Provisioning" AND
2. "did not finish in the allotted time"

This is specific enough to avoid false positives with other failure types.

### State Transitions

Normal flow:
```
Creating → Running → Succeeded
```

With OS provisioning timeout:
```
Creating → Running → Conflict (OS timeout) → potentially still → Succeeded
                               ↓
                         (Continue waiting)
```

Without fix: Fails at "Conflict"
With fix: Continues waiting, may reach "Succeeded"
