# Timeout Behavior: Before vs After Fix

## The Problem

User experiencing VM deployment failures at **600 seconds** with error:
```
Deployment Failed: Conflict - OS Provisioning for VM 'toutb9fc62' did not finish 
in the allotted time. The VM may still finish provisioning successfully.
```

## Timeline Comparison

### BEFORE FIX

```
Timeline (seconds):
0     100   200   300   400   500   600   700   800   900   1000  1100  1200
|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|
                                     ↓ waited 405s
                                     (still running)
                                                    ↓ 600s: Azure OS timeout
                                                    ❌ IMMEDIATE FAILURE
                                                       "Conflict - OS Provisioning timeout"
                                                       
                                                       Jenkins timeout (1200s)
                                                       never reached →

Result: FAILED after 600 seconds
Reason: Azure OS provisioning timeout
Jenkins config (1200s): IGNORED
```

### AFTER FIX

```
Timeline (seconds):
0     100   200   300   400   500   600   700   800   900   1000  1100  1200
|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|
                                     ↓ waited 405s
                                     (still running)
                                                    ↓ 600s: Azure OS timeout
                                                    ⚠️  WARNING logged
                                                       "Azure OS timeout detected"
                                                       "Continuing to wait..."
                                                       
                                                       ↓ Continue polling...
                                                       
                                                          Possible Outcome 1:
                                                          ✅ VM completes (800-1200s)
                                                          
                                                                              ↓ 1200s
                                                                              Possible Outcome 2:
                                                                              ❌ Jenkins timeout
                                                                              Clear error message

Result: 
- Best case: VM completes between 600-1200s ✅
- Worst case: Fails at 1200s with better error message
- Either way: Uses FULL configured timeout
```

## Code Logic

### Detection

```java
boolean isOsProvisioningTimeout = finalStatusMessage != null 
        && finalStatusMessage.contains("OS Provisioning") 
        && finalStatusMessage.contains("did not finish in the allotted time");
```

### Action

```java
if (isOsProvisioningTimeout) {
    // Log warning but DON'T fail
    LOGGER.log(Level.WARNING, 
        "Azure OS provisioning timeout detected after {0} seconds. "
        + "This is Azure-side timeout (~10 min), not Jenkins timeout. "
        + "Continuing to wait up to {1} seconds total...", 
        waitedSeconds, timeoutInSeconds);
    // Continue to next polling iteration
} else {
    // Other failures: fail immediately (unchanged)
    throw AzureCloudException.create(...);
}
```

## Log Output Comparison

### BEFORE FIX

```
INFO: Waiting for deployment tout0204130819525 with VM toutb9fc62...
      Deployment timeout: 1200 seconds (20 minutes)
INFO: Deployment not yet finished (Running)...waited 60 seconds
INFO: Deployment not yet finished (Running)...waited 120 seconds
...
INFO: Deployment not yet finished (Running)...waited 405 seconds
...
ERROR: Failure creating provisioned agent 'toutb9fc62'
       AzureCloudException: Deployment Failed: Conflict - 
       OS Provisioning for VM 'toutb9fc62' did not finish in the allotted time.
       
❌ FAILED AT 600 SECONDS (Azure timeout)
   Jenkins timeout (1200s) never reached
```

### AFTER FIX

```
INFO: Waiting for deployment tout0204130819525 with VM toutb9fc62...
      Deployment timeout: 1200 seconds (20 minutes)
INFO: Deployment not yet finished (Running)...waited 60 seconds
INFO: Deployment not yet finished (Running)...waited 120 seconds
...
INFO: Deployment not yet finished (Running)...waited 405 seconds
...
INFO: Deployment not yet finished (Running)...waited 600 seconds

⚠️  WARNING: Azure OS provisioning timeout detected for VM toutb9fc62 after 600 seconds.
             This is an Azure-side timeout (typically ~10 minutes), not Jenkins timeout.
             Azure reports the VM may still finish provisioning.
             Continuing to wait up to Jenkins timeout (1200 seconds total).
             Error: Conflict - OS Provisioning...did not finish in the allotted time

INFO: Deployment not yet finished (Running)...waited 660 seconds
INFO: Deployment not yet finished (Running)...waited 720 seconds
...

EITHER:
✅ INFO: VM available: toutb9fc62
   (VM completed between 600-1200s)

OR:
❌ ERROR: Deployment failed: Jenkins deployment timeout reached (1200 seconds).
          The VM deployment did not complete within the configured timeout.
          This may be due to slow VM provisioning or Azure OS provisioning issues.
          Consider: 1) Increasing deploymentTimeout, 2) Using properly prepared image...
```

## Key Differences

| Aspect | Before Fix | After Fix |
|--------|-----------|-----------|
| **Failure time** | 600s (Azure timeout) | Up to 1200s (Jenkins timeout) |
| **Uses Jenkins config?** | ❌ No (ignores 1200s setting) | ✅ Yes (respects 1200s setting) |
| **VM success chance** | 0% after 600s | Possible if completes 600-1200s |
| **Error clarity** | "Deployment Failed" | "Azure OS timeout" vs "Jenkins timeout" |
| **Actionable guidance** | Minimal | Specific troubleshooting steps |
| **Logging** | Basic | Detailed with timeout distinction |

## Why This Matters

### Real-World Scenarios

**Scenario 1: Slow but functional image**
- Image takes 15 minutes to fully provision
- Before: ❌ Fails at 10 min (Azure timeout)
- After: ✅ Succeeds at 15 min (within Jenkins 20 min timeout)

**Scenario 2: Azure platform delay**
- Azure experiencing temporary slowness
- VM would succeed with a bit more time
- Before: ❌ Fails immediately at Azure timeout
- After: ✅ Might succeed with extended wait

**Scenario 3: Broken image**
- Image truly broken, never completes
- Before: ❌ Fails at 600s with unclear message
- After: ❌ Still fails but at 1200s with clear guidance on how to fix

## Configuration

The fix works with existing Jenkins configuration:

```
Cloud Configuration → Advanced → Deployment Timeout
Default: 1200 seconds (20 minutes)
Recommended: 1200-2400 seconds
```

No configuration changes needed - fix automatically:
1. Detects Azure OS timeout at ~600s
2. Continues waiting up to Jenkins configured timeout
3. Provides clear logging throughout

## Summary

**The fix allows VMs to use the FULL configured Jenkins timeout (1200s), 
rather than failing early at Azure's hardcoded OS provisioning timeout (600s).**

This is exactly what Azure's error message suggests: 
"The VM may still finish provisioning successfully. Please check provisioning state later."

We now check "later" (up to Jenkins timeout) instead of giving up immediately.
