# Testing Guide: Diagnostic Logging and Configurable Retry Interval

## What Was Implemented

### 1. Comprehensive Diagnostic Logging
Added extensive logging to help identify the source of the 10-minute timeout, since **Azure support confirms there is NO Azure-defined 10-minute timeout**.

### 2. Configurable Max Retry Interval
Made the previously hardcoded 10-minute retry backoff interval configurable through the UI.

## How to Build and Test

### Build the Plugin

```bash
cd /home/runner/work/azure-vm-agents-plugin/azure-vm-agents-plugin
mvn clean package -DskipTests
```

The plugin will be in: `target/azure-vm-agents.hpi`

### Install the Plugin

1. Go to Jenkins → Manage Jenkins → Manage Plugins
2. Click "Advanced" tab
3. Under "Upload Plugin", select the `.hpi` file
4. Upload and restart Jenkins

## Testing the Diagnostic Logging

### Setup

1. Configure an Azure VM Cloud in Jenkins
2. In Advanced settings:
   - Set **Deployment Timeout**: 1200 (or higher)
   - Set **Max Retry Interval**: 600 (or your desired value)
3. Configure a template that will take >10 minutes to provision

### Run a Test Deployment

Trigger a VM provisioning and monitor the logs.

### What to Look For in Logs

#### 1. Start of Deployment
```
[DIAGNOSTIC] Starting deployment wait for deploy-20260204... with VM vmname-0. 
Deployment timeout: 1200 seconds (20 minutes), max polling attempts: 240, 
start time: Tue Feb 04 13:00:00 UTC 2026
```

**Check:** Confirms configured timeout value

#### 2. Progress Logging
```
[DIAGNOSTIC] Deployment deploy-20260204: Elapsed time 300 seconds (5 minutes), tries left: 180
[DIAGNOSTIC] Deployment deploy-20260204: Elapsed time 590 seconds (9 minutes), tries left: 122
[DIAGNOSTIC] Deployment deploy-20260204: Elapsed time 595 seconds (9 minutes), tries left: 121
[DIAGNOSTIC] Deployment deploy-20260204: Elapsed time 600 seconds (10 minutes), tries left: 120
[DIAGNOSTIC] Deployment deploy-20260204: Elapsed time 605 seconds (10 minutes), tries left: 119
[DIAGNOSTIC] Deployment deploy-20260204: Elapsed time 610 seconds (10 minutes), tries left: 118
```

**Check:** 
- Are logs appearing around 590-610 second mark?
- Does polling continue past 600 seconds?
- What happens at exactly 600 seconds?

#### 3. When Error Occurs
```
[DIAGNOSTIC] Deployment deploy-20260204 failed at 598 seconds. 
State: Conflict, Status code: Conflict, VM: vmname-0, 
Type: Microsoft.Compute/virtualMachines, 
Message: Conflict - OS Provisioning for VM 'vmname-0' did not finish in the allotted time...
```

**Critical Information:**
- **Exact timing**: At what second does the error occur?
- **State**: What deployment state? (Conflict, Failed, Running, etc.)
- **Status code**: What's the status code?
- **Full message**: Complete error message from Azure

#### 4. OS Provisioning Timeout Pattern Detection
```
[DIAGNOSTIC] Detected OS provisioning timeout pattern at 600 seconds for VM vmname-0. 
Azure support confirms NO Azure-defined 10-minute timeout exists. 
This may be HTTP client timeout or other plugin timeout. 
Continuing to wait up to Jenkins timeout (1200 seconds total). 
Error: Conflict - OS Provisioning...did not finish in the allotted time
```

**Check:**
- Is this message logged?
- At what second is it detected?
- Does polling continue after this?

#### 5. Exception Details
```
[DIAGNOSTIC] Unexpected exception for deployment deploy-20260204 at 605 seconds. 
Exception type: java.net.SocketTimeoutException, 
Message: Read timed out, 
Cause: none

[DIAGNOSTIC] Full stack trace:
java.net.SocketTimeoutException: Read timed out
    at java.net.SocketInputStream.socketRead0(Native Method)
    at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
    at java.net.SocketInputStream.read(SocketInputStream.java:171)
    ...
```

**Critical Information:**
- **Exception type**: Is it `SocketTimeoutException`? `HttpTimeoutException`? `AzureException`?
- **When thrown**: At what second?
- **Stack trace**: Where did it originate? HTTP client? Azure SDK?

#### 6. Final Timeout (if VM doesn't complete)
```
[DIAGNOSTIC] Deployment deploy-20260204 reached Jenkins timeout. 
Configured timeout: 1200s, Actual elapsed: 1205s
```

**Check:**
- Did deployment reach Jenkins timeout?
- Is actual elapsed time close to configured timeout?

## Analysis: What to Report Back

### Scenario 1: HTTP Client Timeout (Most Likely)

**Expected logs:**
```
[DIAGNOSTIC] Elapsed time 600 seconds
[DIAGNOSTIC] Unexpected exception at 603 seconds
Exception type: java.net.SocketTimeoutException
Message: Read timed out
Stack trace: ...HttpClient...
```

**Conclusion:** HTTP client has 10-minute timeout

**Solution:** Need to configure HTTP client timeout (separate issue)

### Scenario 2: Azure API Error

**Expected logs:**
```
[DIAGNOSTIC] Deployment failed at 598 seconds
State: Conflict
Message: OS Provisioning...did not finish
[DIAGNOSTIC] Detected OS provisioning timeout pattern
```

**Conclusion:** Azure returns error at ~600s

**Solution:** Continue waiting logic (already implemented)

### Scenario 3: Jenkins Timeout Working Correctly

**Expected logs:**
```
[DIAGNOSTIC] Elapsed time 600 seconds
[DIAGNOSTIC] Elapsed time 900 seconds
[DIAGNOSTIC] Elapsed time 1200 seconds
[DIAGNOSTIC] Deployment reached Jenkins timeout. Configured: 1200s, Actual: 1203s
```

**Conclusion:** Timeout is working as expected

**Solution:** May need to increase timeout or fix image

## Testing Max Retry Interval Configuration

### Test Retry Backoff

1. Configure cloud with **Max Retry Interval**: 300 (5 minutes)
2. Create a template that will fail to provision
3. Trigger provisioning
4. Watch for retry attempts

**Expected behavior:**
- 1st failure: wait 10 seconds
- 2nd failure: wait 20 seconds
- 3rd failure: wait 40 seconds
- 4th failure: wait 80 seconds
- 5th failure: wait 160 seconds
- 6th failure: wait 300 seconds (capped at configured max)
- 7th+ failures: wait 300 seconds

**Log entries to look for:**
```
Template verification failed
Retry interval extended to: 10000ms
Retry interval extended to: 20000ms
Retry interval extended to: 40000ms
Retry interval extended to: 80000ms
Retry interval extended to: 160000ms
Retry interval extended to: 300000ms (capped at max)
```

## Key Questions to Answer

Based on the diagnostic logs, please provide:

1. **Exact timing of failure:**
   - At what second does the error/exception occur?
   - Is it exactly 600? Earlier? Later?

2. **Exception details:**
   - What is the exception class name?
   - What is the complete error message?
   - What does the stack trace show?

3. **Deployment state:**
   - What state is the deployment in when error occurs?
   - Is it "Conflict"? "Failed"? "Running"?

4. **Polling behavior:**
   - Do you see logs at 590, 595, 600, 605, 610 seconds?
   - Does polling continue past 600 seconds?
   - Or does it stop at 600 seconds?

5. **Error source:**
   - Does the error message come from Azure API?
   - Or does it come from HTTP client?
   - Or from somewhere else in the stack?

## Expected Deliverables

Please provide:

1. **Complete Jenkins log** from deployment start to failure
2. **All [DIAGNOSTIC] log entries** (grep for "[DIAGNOSTIC]")
3. **Full stack trace** of any exceptions
4. **Screenshot** of Advanced settings showing configured timeouts
5. **Answers** to the key questions above

## What This Will Tell Us

With this information, we can definitively determine:

- ✅ Is the timeout from Azure, HTTP client, or something else?
- ✅ At what exact time does the timeout occur?
- ✅ Can we extend/configure this timeout?
- ✅ Is the current fix (continue waiting) working correctly?
- ✅ What additional changes (if any) are needed?

## Next Steps After Testing

Based on test results, we may need to:

1. **If HTTP client timeout**: Configure HTTP client with longer timeout
2. **If Azure API error**: Current fix should handle it (continue waiting)
3. **If plugin code**: Identify and fix the source
4. **If Jenkins timeout**: Working as expected, may need to increase value

Thank you for testing!
