# Test Compilation Fixes

## Problem

After adding the `maxRetryIntervalSeconds` parameter to the `AzureVMCloud` constructor, all test files that instantiate `AzureVMCloud` directly failed to compile because they were passing only 8 parameters instead of the required 9.

## Compilation Errors

### Error 1: AzureVMCloudTest.java
```
[ERROR] constructor AzureVMCloud cannot be applied to given types;
required: String,String,String,String,String,String,String,String,List
found:    String,String,String,String,String,String,String,String (8 params)
```

### Error 2: ITAzureVMCloud.java
Same error at two locations (lines 43 and 84).

## Root Cause

The `AzureVMCloud` constructor signature was updated from 8 to 9 parameters:

```java
// Constructor signature (9 parameters)
public AzureVMCloud(
        String name,                        // 1
        String azureCredentialsId,          // 2
        String maxVirtualMachinesLimit,     // 3
        String deploymentTimeout,           // 4
        String maxRetryIntervalSeconds,     // 5 ← NEW PARAMETER
        String resourceGroupReferenceType,  // 6
        String newResourceGroupName,        // 7
        String existingResourceGroupName,   // 8
        List<AzureVMAgentTemplate> vmTemplates) // 9
```

But test files were still using the old 8-parameter calls.

## Solution

Added `null` as the 5th parameter (`maxRetryIntervalSeconds`) in all test constructor calls. When `null` is passed, the constructor uses the default value from `Constants.DEFAULT_MAX_RETRY_INTERVAL_SEC` (600 seconds).

## Changes Made

### 1. AzureVMCloudTest.java (Line 288)

**Before:**
```java
private static AzureVMCloud mkInstance(int maxVMsLimitForCloud) {
    return new AzureVMCloud(null, null, Integer.toString(maxVMsLimitForCloud), 
                            null, null, null, null, null);
    // 8 parameters ↑
}
```

**After:**
```java
private static AzureVMCloud mkInstance(int maxVMsLimitForCloud) {
    return new AzureVMCloud(null, null, Integer.toString(maxVMsLimitForCloud), 
                            null, null, null, null, null, null);
    // 9 parameters ↑ (added null for maxRetryIntervalSeconds)
}
```

### 2. ITAzureVMCloud.java (Line 43)

**Before:**
```java
AzureVMCloud cloudMock = spy(
    new AzureVMCloud("", "xyz", "42", "0", "new", 
                     testEnv.azureResourceGroup, null, null));
// 8 parameters
```

**After:**
```java
AzureVMCloud cloudMock = spy(
    new AzureVMCloud("", "xyz", "42", "0", null, "new", 
                     testEnv.azureResourceGroup, null, null));
// 9 parameters (added null for maxRetryIntervalSeconds)
```

### 3. ITAzureVMCloud.java (Line 84)

**Before:**
```java
AzureVMCloud cloudMock = spy(
    new AzureVMCloud("", credentialsId, "42", "30", "new", 
                     testEnv.azureResourceGroup, null, null));
// 8 parameters
```

**After:**
```java
AzureVMCloud cloudMock = spy(
    new AzureVMCloud("", credentialsId, "42", "30", null, "new", 
                     testEnv.azureResourceGroup, null, null));
// 9 parameters (added null for maxRetryIntervalSeconds)
```

## Files Modified

1. `src/test/java/com/microsoft/azure/vmagent/AzureVMCloudTest.java`
   - Line 288: Helper method `mkInstance()`

2. `src/test/java/com/microsoft/azure/vmagent/ITAzureVMCloud.java`
   - Line 43: Test for failed deployment scenario
   - Line 84: Test for VM provisioning

## Impact

### Test Behavior
- Tests now use the default `maxRetryIntervalSeconds` value (600 seconds)
- No change to actual test logic - tests still validate the same behavior
- Tests remain focused on their original purpose

### Verification
All test constructor calls verified:
```bash
$ grep -r "new AzureVMCloud(" src/test --include="*.java"
# Found and fixed all 3 instances
```

## Result

✅ **All test files compile successfully**  
✅ **All constructor calls use correct 9-parameter signature**  
✅ **Tests use default retry interval (consistent with production code)**  
✅ **No behavioral changes to tests**

## Related Changes

This fix is part of the larger change to make the retry interval configurable:
- Added `maxRetryIntervalSeconds` parameter to `AzureVMCloud` constructor
- Updated `AzureVMCloudBuilder` to support the new parameter
- Fixed all test files to use the updated constructor

Total instances fixed: **3 test constructor calls**
