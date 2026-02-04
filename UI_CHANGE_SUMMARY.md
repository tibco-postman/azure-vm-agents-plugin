# UI Changes - Deployment Timeout

## Before (Main Configuration Section)

```
┌─────────────────────────────────────────────┐
│ Azure VM Agents Cloud Configuration        │
├─────────────────────────────────────────────┤
│ Cloud Name:           [azure-cloud       ] │
│ Azure Credentials:    [Select credentials▼] │
│ Max VM Limit:         [10                ] │
│ Deployment Timeout:   [1200              ] │  ← Was here
│ Custom Tag:           [Add tag           ] │
│ Resource Group Name:  ⚪ Create new         │
│                       ⚫ Use existing        │
│                          [my-rg          ] │
│                                             │
│ [Verify Configuration]                      │
└─────────────────────────────────────────────┘
```

## After (Advanced Section)

```
┌─────────────────────────────────────────────┐
│ Azure VM Agents Cloud Configuration        │
├─────────────────────────────────────────────┤
│ Cloud Name:           [azure-cloud       ] │
│ Azure Credentials:    [Select credentials▼] │
│ Max VM Limit:         [10                ] │
│ Custom Tag:           [Add tag           ] │
│ Resource Group Name:  ⚪ Create new         │
│                       ⚫ Use existing        │
│                          [my-rg          ] │
│                                             │
│ [▶ Advanced]                                │  ← Click to expand
│                                             │
│ [Verify Configuration]                      │
└─────────────────────────────────────────────┘

When Advanced is clicked:

┌─────────────────────────────────────────────┐
│ [▼ Advanced]                                │
│ ┌─────────────────────────────────────────┐ │
│ │ Deployment Timeout:  [1200            ] │ │  ← Now here
│ │                                         │ │
│ │ ⓘ Specify the number of seconds to     │ │
│ │   wait for a VM deployment to complete. │ │
│ │   Default: 1200 seconds (20 minutes)    │ │
│ └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

## Benefits

1. **Cleaner main view** - Less cluttered configuration screen
2. **Better organization** - Advanced settings grouped together
3. **Still accessible** - One click to expand Advanced section
4. **Backward compatible** - Existing values preserved
5. **Better help** - Enhanced tooltip with defaults and recommendations

## Configuration Methods

### Method 1: Jenkins UI
1. Manage Jenkins → Nodes → Clouds → [Your Cloud]
2. Click "Advanced" button
3. Set "Deployment Timeout" (in seconds)

### Method 2: Configuration as Code (JCasC)
```yaml
jenkins:
  clouds:
    - azureVM:
        name: "my-azure-cloud"
        azureCredentialsId: "azure-cred"
        deploymentTimeout: 1200  # Still configurable via YAML
        maxVirtualMachinesLimit: 10
        resourceGroupReferenceType: "existing"
        existingResourceGroupName: "my-rg"
```

### Method 3: Groovy Script
```groovy
import com.microsoft.azure.vmagent.AzureVMCloud

def cloud = Jenkins.instance.clouds.find { it.name == 'my-azure-cloud' }
println "Current deployment timeout: ${cloud.deploymentTimeout} seconds"
```
