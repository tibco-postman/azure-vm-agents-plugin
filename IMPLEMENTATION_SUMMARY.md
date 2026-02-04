# Summary: Diagnostic Logging and Configurable Retry Interval

## Changes Implemented

### 1. Comprehensive Diagnostic Logging (Primary Feature)

**Purpose:** Identify the source of the 10-minute timeout, since Azure support confirms **NO Azure-defined 10-minute timeout exists**.

**Key Logging Added:**

```java
// Start time and configuration
[DIAGNOSTIC] Starting deployment wait for {deployment} with VM {vm}. 
Deployment timeout: 1200 seconds (20 minutes), max polling attempts: 240, 
start time: {timestamp}

// Progress at critical times (every 5 min, and specifically around 600s)
[DIAGNOSTIC] Deployment {name}: Elapsed time 590 seconds (9 minutes), tries left: 122
[DIAGNOSTIC] Deployment {name}: Elapsed time 600 seconds (10 minutes), tries left: 120
[DIAGNOSTIC] Deployment {name}: Elapsed time 610 seconds (10 minutes), tries left: 118

// When deployment fails
[DIAGNOSTIC] Deployment {name} failed at {X} seconds. 
State: {state}, Status code: {code}, VM: {vm}, Type: {type}, Message: {msg}

// OS provisioning timeout detection (updated with new knowledge)
[DIAGNOSTIC] Detected OS provisioning timeout pattern at {X} seconds for VM {vm}. 
Azure support confirms NO Azure-defined 10-minute timeout exists. 
This may be HTTP client timeout or other plugin timeout. 
Continuing to wait up to Jenkins timeout ({Y} seconds total).

// Exception details
[DIAGNOSTIC] Unexpected exception for deployment {name} at {X} seconds. 
Exception type: {class}, Message: {msg}, Cause: {cause}
[DIAGNOSTIC] Full stack trace: {trace}

// Final timeout
[DIAGNOSTIC] Deployment {name} reached Jenkins timeout. 
Configured timeout: {X}s, Actual elapsed: {Y}s
```

**What This Reveals:**
- Exact second when timeout/error occurs
- Exception type and stack trace
- Deployment state at failure
- Whether polling continues past 600s
- Source of error (Azure API? HTTP client? Plugin?)

### 2. Configurable Max Retry Interval (Secondary Feature)

**Purpose:** Allow users to configure the previously hardcoded 10-minute retry backoff interval.

**Implementation:**

**Before:**
```java
// ProvisionStrategy.java
private static final long MAX_INTERVAL = 10 * 60 * 1000; // Hardcoded
```

**After:**
```java
// ProvisionStrategy.java
private final long maxInterval; // Configurable via constructor

public ProvisionStrategy(int maxIntervalSeconds) {
    this.maxInterval = maxIntervalSeconds * 1000L;
}

// AzureVMCloud.java
private final int maxRetryIntervalSeconds; // Field
// Constructor parameter, getter method

// Constants.java
public static final int DEFAULT_MAX_RETRY_INTERVAL_SEC = 600;
```

**UI:**
- Added "Max Retry Interval" field in Advanced section
- Default: 600 seconds (10 minutes)
- Help file explaining what it controls

**What This Controls:**
- Exponential backoff after provisioning failures
- 1st failure: 10s → 2nd: 20s → 3rd: 40s → ... → caps at configured max
- **NOT** deployment timeout (that's separate)

## Files Modified

### Code Changes
1. ✅ `src/main/java/com/microsoft/azure/vmagent/AzureVMCloud.java`
   - Added diagnostic logging with timing, exceptions, state tracking
   - Added `maxRetryIntervalSeconds` field
   - Constructor, getter, descriptor method

2. ✅ `src/main/java/com/microsoft/azure/vmagent/ProvisionStrategy.java`
   - Made MAX_INTERVAL configurable via constructor
   - Removed static final hardcoded value

3. ✅ `src/main/java/com/microsoft/azure/vmagent/AzureVMAgentTemplate.java`
   - Pass maxRetryIntervalSeconds to ProvisionStrategy constructor

4. ✅ `src/main/java/com/microsoft/azure/vmagent/util/Constants.java`
   - Added `DEFAULT_MAX_RETRY_INTERVAL_SEC = 600`

### UI Changes
5. ✅ `src/main/resources/com/microsoft/azure/vmagent/AzureVMCloud/config.jelly`
   - Added "Max Retry Interval" field in Advanced section
   - Updated validation button parameters

6. ✅ `src/main/webapp/help-maxRetryIntervalSeconds.html`
   - Created help file explaining retry interval
   - Clarified difference from deployment timeout
   - Noted Azure has NO 10-minute timeout

### Documentation
7. ✅ `TESTING_GUIDE.md`
   - Complete testing instructions
   - What to look for in logs
   - Analysis scenarios
   - Key questions to answer
   - Expected deliverables

8. ✅ `EXCEPTION_HANDLING_ANALYSIS.md`
   - Detailed exception flow analysis
   - Stack trace explanation

9. ✅ `AZURE_OS_PROVISIONING_TIMEOUT_FIX.md`
   - Original fix documentation

10. ✅ `TIMEOUT_COMPARISON.md`
    - Visual before/after comparison

11. ✅ `DEPLOYMENT_TIMEOUT_CHANGES.md`
    - General timeout improvements

## Understanding the 10-Minute Timeout

### Previous Understanding (INCORRECT)
- Believed Azure had 10-minute OS provisioning timeout
- Implemented fix to continue waiting past Azure's timeout

### Current Understanding (CORRECT)
- **Azure support confirms: NO Azure-defined 10-minute timeout**
- Timeout must come from plugin code or dependencies
- Likely candidates:
  1. HTTP client default timeout (most likely)
  2. Some other plugin code
  3. Azure SDK configuration

### How Diagnostic Logging Helps

The new logging will definitively identify:

**If HTTP client timeout:**
```
[DIAGNOSTIC] Unexpected exception at 603 seconds
Exception type: java.net.SocketTimeoutException
Stack trace: ...HttpClient...timeout...
```

**If Azure API returns error:**
```
[DIAGNOSTIC] Deployment failed at 598 seconds
State: Conflict, Message: OS Provisioning...
```

**If plugin code issue:**
```
[DIAGNOSTIC] Exception at XXX seconds
Exception type: {specific class}
Stack trace: {plugin code}
```

## Next Steps

### Immediate: Testing
1. User builds plugin with new logging
2. User runs test deployment
3. User collects diagnostic logs
4. User reports findings:
   - Exact timing of failure
   - Exception type and stack trace
   - Deployment state
   - Whether polling continues past 600s

### Based on Test Results:

**Scenario 1: HTTP Client Timeout**
- Investigate HTTP client configuration
- Look into `HttpClientRetriever` from azuresdk plugin
- May need to configure longer HTTP timeout
- Or create custom HTTP client

**Scenario 2: Azure API Error**
- Current fix already handles this (continue waiting)
- Verify fix is working as expected
- May need to refine error detection

**Scenario 3: Other Plugin Code**
- Identify specific code causing timeout
- Implement targeted fix
- Add configuration if needed

## Configuration

### Deployment Timeout (existing)
```yaml
jenkins:
  clouds:
    - azureVM:
        deploymentTimeout: 1200  # How long to wait for VM deployment
```

### Max Retry Interval (new)
```yaml
jenkins:
  clouds:
    - azureVM:
        maxRetryIntervalSeconds: 600  # Max backoff between retries after failures
```

### Via UI
Both settings now in: Cloud Configuration → Advanced section

## Summary

### What We Know
- ✅ Azure has NO 10-minute timeout (confirmed by Azure support)
- ✅ Timeout is coming from plugin or dependencies
- ✅ Previous fix was based on incorrect assumption

### What We Added
- ✅ Extensive diagnostic logging to identify timeout source
- ✅ Configurable retry interval (was hardcoded to 10 min)
- ✅ Comprehensive testing guide
- ✅ Updated documentation reflecting correct understanding

### What We Need
- 🔍 Test results with diagnostic logs
- 🔍 Exception details and stack traces
- 🔍 Exact timing of timeout

### What We'll Do Next
- 🎯 Analyze test results
- 🎯 Identify actual timeout source
- 🎯 Implement targeted fix
- 🎯 Verify fix works correctly

## Build and Test

```bash
# Build
cd /home/runner/work/azure-vm-agents-plugin/azure-vm-agents-plugin
mvn clean package -DskipTests

# Output
target/azure-vm-agents.hpi

# Install in Jenkins
Manage Jenkins → Manage Plugins → Advanced → Upload Plugin

# Test and collect logs
See TESTING_GUIDE.md for details
```

The diagnostic logging will reveal the truth about where the 10-minute timeout originates!
