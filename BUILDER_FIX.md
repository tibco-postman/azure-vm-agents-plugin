# Builder Compilation Error Fix

## Problem

After adding the `maxRetryIntervalSeconds` parameter to the `AzureVMCloud` constructor, the code failed to compile because the `AzureVMCloudBuilder` was not updated to pass this new parameter.

### Compilation Error
```
[ERROR] constructor AzureVMCloud in class com.microsoft.azure.vmagent.AzureVMCloud cannot be applied to given types;
  required: java.lang.String,java.lang.String,java.lang.String,java.lang.String,java.lang.String,java.lang.String,java.lang.String,java.lang.String,java.util.List
  found:    java.lang.String,java.lang.String,java.lang.String,java.lang.String,java.lang.String,java.lang.String,java.lang.String,java.util.List
  reason: actual and formal argument lists differ in length
```

**Location:** `AzureVMCloudBuilder.java` line 108 (in the `build()` method)

## Root Cause

The `AzureVMCloud` constructor signature was changed from:
```java
// Old - 8 parameters
public AzureVMCloud(
        String name,
        String azureCredentialsId,
        String maxVirtualMachinesLimit,
        String deploymentTimeout,
        String resourceGroupReferenceType,
        String newResourceGroupName,
        String existingResourceGroupName,
        List<AzureVMAgentTemplate> vmTemplates)
```

To:
```java
// New - 9 parameters (added maxRetryIntervalSeconds)
public AzureVMCloud(
        String name,
        String azureCredentialsId,
        String maxVirtualMachinesLimit,
        String deploymentTimeout,
        String maxRetryIntervalSeconds,  // ← NEW PARAMETER
        String resourceGroupReferenceType,
        String newResourceGroupName,
        String existingResourceGroupName,
        List<AzureVMAgentTemplate> vmTemplates)
```

But `AzureVMCloudBuilder.build()` was still calling the constructor with only 8 parameters.

## Solution

Updated `AzureVMCloudBuilder` to include the new parameter:

### 1. Added Field
```java
private String maxRetryIntervalSeconds;
```

### 2. Default Constructor
```java
public AzureVMCloudBuilder() {
    maxVirtualMachinesLimit = "10";
    deploymentTimeout = "1200";
    maxRetryIntervalSeconds = "600";  // ← NEW: Default 10 minutes
    resourceGroupReferenceType = "new";
    vmTemplates = new ArrayList<>();
}
```

### 3. Copy Constructor
```java
public AzureVMCloudBuilder(AzureVMCloud cloud) {
    cloudName = cloud.getCloudName();
    azureCredentialsId = cloud.getAzureCredentialsId();
    maxVirtualMachinesLimit = String.valueOf(cloud.getMaxVirtualMachinesLimit());
    deploymentTimeout = String.valueOf(cloud.getDeploymentTimeout());
    maxRetryIntervalSeconds = String.valueOf(cloud.getMaxRetryIntervalSeconds());  // ← NEW
    resourceGroupReferenceType = cloud.getResourceGroupReferenceType();
    newResourceGroupName = cloud.getNewResourceGroupName();
    existingResourceGroupName = cloud.getExistingResourceGroupName();
    vmTemplates = new ArrayList<>();
    vmTemplates.addAll(cloud.getVmTemplates());
}
```

### 4. Fluent API Method
```java
public AzureVMCloudBuilder withMaxRetryIntervalSeconds(String maxRetryIntervalSeconds) {
    this.maxRetryIntervalSeconds = maxRetryIntervalSeconds;
    return this;
}
```

### 5. Updated build() Method
```java
public AzureVMCloud build() {
    return new AzureVMCloud(
            StringUtils.defaultString(cloudName),
            StringUtils.defaultString(azureCredentialsId),
            maxVirtualMachinesLimit,
            deploymentTimeout,
            maxRetryIntervalSeconds,  // ← NEW PARAMETER
            resourceGroupReferenceType,
            StringUtils.defaultString(newResourceGroupName),
            StringUtils.defaultString(existingResourceGroupName),
            vmTemplates);
}
```

## Benefits

1. **Fixes compilation** - Builder now passes correct number of parameters
2. **Backward compatible** - Existing code using builder will use default value (600 seconds)
3. **Fluent API** - Supports configuration via `withMaxRetryIntervalSeconds()`
4. **Consistent defaults** - Default "600" matches `Constants.DEFAULT_MAX_RETRY_INTERVAL_SEC`
5. **Copy support** - Copy constructor properly extracts value from existing cloud

## Files Modified

- `src/main/java/com/microsoft/azure/vmagent/builders/AzureVMCloudBuilder.java`
  - Line 21: Added field
  - Line 34: Default constructor initialization
  - Line 44: Copy constructor extraction
  - Lines 74-77: Fluent API method
  - Line 121: Pass to constructor

## Testing

Code now compiles successfully. The builder can be used in three ways:

```java
// 1. Default value (600 seconds)
AzureVMCloud cloud = new AzureVMCloudBuilder()
    .withCloudName("test")
    .build();

// 2. Custom value
AzureVMCloud cloud = new AzureVMCloudBuilder()
    .withCloudName("test")
    .withMaxRetryIntervalSeconds("900")
    .build();

// 3. Copy from existing
AzureVMCloud newCloud = new AzureVMCloudBuilder(existingCloud)
    .withMaxRetryIntervalSeconds("1200")
    .build();
```
