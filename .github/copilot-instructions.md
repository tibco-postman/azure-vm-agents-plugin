# Azure VM Agents Plugin - Copilot Instructions

## Repository Overview

This is a **Jenkins plugin** that provisions Jenkins build agents on Azure Virtual Machines using ARM templates. The plugin supports both Windows and Linux agents with various customization options.

**Project Type:** Jenkins Plugin (HPI/JPI packaging)  
**Language:** Java  
**Build System:** Apache Maven  
**Size:** ~2.3MB, 58 Java source files (~14,000 lines of code)  
**Target:** Jenkins 2.516.3+ with Java 17+

## Critical Build Information

### Prerequisites
- **Java:** 17 or 21 (as per Jenkinsfile configurations)
- **Maven:** 3.9.x or higher
- **Network:** Requires access to `repo.jenkins-ci.org` and Maven Central

### Maven Configuration
The project uses Maven with Jenkins-specific extensions:
- `.mvn/maven.config` defines incremental build profiles
- `.mvn/extensions.xml` includes `git-changelist-maven-extension` for versioning
- Parent POM: `org.jenkins-ci.plugins:plugin:6.2122.v70b_7b_f659d72`

### Build Commands

**IMPORTANT:** This project requires internet access to Jenkins and Maven repositories. In network-restricted environments, the build will fail with "No address associated with hostname" errors.

#### Standard Build Sequence (with network access):
```bash
# Clean the project
mvn clean

# Compile and validate (includes checkstyle)
mvn verify

# Run unit tests only (faster)
mvn test

# Run integration tests
mvn verify -Pit

# Package the plugin (creates .hpi file)
mvn package

# Skip tests during development
mvn package -DskipTests

# Run checkstyle separately
mvn checkstyle:check
```

#### Common Build Issues:

1. **Network Errors:** If you see `repo.jenkins-ci.org: No address associated with hostname`, the build environment lacks internet access. This is expected in sandboxed environments. Document this limitation rather than attempting workarounds.

2. **Checkstyle Violations:** The build runs checkstyle during the `validate` phase. Configuration is in `checkstyle.xml`. Some files are explicitly excluded (see pom.xml lines 190-194):
   - `com/microsoft/azure/vmagent/Messages.java`
   - `com/microsoft/azure/vmagent/AzureVMManagementServiceDelegate.java`
   - `com/microsoft/azure/vmagent/remote/AzureVMAgentSSHLauncher.java`

3. **Test Execution:** Tests use parallel execution. Configuration in pom.xml:
   - Surefire: `forkCount=1C`, parallel=all, unlimited threads
   - Failsafe: `forkCount=3`, `reuseForks=true`, `threadCountMethods=7`

### Validation Steps

**Always run these in order before submitting changes:**

1. **Checkstyle:** `mvn checkstyle:check` (runs automatically in `validate` phase)
2. **Compile:** `mvn compile` 
3. **Tests:** `mvn test` (unit tests) or `mvn verify` (includes integration tests)
4. **Package:** `mvn package` (creates the .hpi plugin file in `target/`)

## Project Structure

### Source Code Layout
```
src/
├── main/
│   ├── java/com/microsoft/azure/vmagent/
│   │   ├── AzureVMCloud.java              # Main cloud provider
│   │   ├── AzureVMAgent.java              # Agent representation
│   │   ├── AzureVMAgentTemplate.java      # Agent template config
│   │   ├── AzureVMManagementServiceDelegate.java  # Azure SDK interactions
│   │   ├── builders/                      # Fluent builders for configuration
│   │   ├── remote/                        # SSH launcher implementations
│   │   │   └── AzureVMAgentSSHLauncher.java
│   │   ├── launcher/                      # Launch mechanisms
│   │   ├── availability/                  # Availability sets/zones
│   │   ├── retry/                         # Retry logic
│   │   ├── util/                          # Utilities
│   │   └── exceptions/                    # Custom exceptions
│   ├── resources/
│   │   ├── com/microsoft/azure/vmagent/   # Jelly UI files and Messages.properties
│   │   ├── scripts/                       # Ubuntu install scripts (Git, Maven, Docker, etc.)
│   │   ├── *.json                         # ARM templates for VM provisioning
│   │   └── index.jelly                    # Plugin metadata
│   └── webapp/                            # Help HTML files (help-*.html)
├── test/
│   ├── java/com/microsoft/azure/vmagent/
│   │   ├── *Test.java                     # Unit tests
│   │   ├── IT*.java                       # Integration tests
│   │   └── test/jcasc/                    # JCasC configuration tests
│   └── resources/                         # Test resources
└── spotbugs/                              # SpotBugs configuration
```

### Key Configuration Files
- **pom.xml** - Maven build configuration, dependencies, plugin configuration
- **checkstyle.xml** - Code style rules (based on Sun checks, max line length: 120)
- **Jenkinsfile** - Jenkins CI pipeline (builds on Linux with JDK 17 and 21)
- **.mvn/maven.config** - Maven flags for incremental builds
- **.github/workflows/cd.yaml** - Continuous delivery workflow
- **.github/workflows/jenkins-security-scan.yml** - Security scanning on PR/push to master

### Root Directory Files
```
.github/              # GitHub workflows and configuration
.mvn/                 # Maven wrapper configuration
docs/                 # Documentation (configure-with-groovy.md, init scripts)
src/                  # Source code
checkstyle.xml        # Checkstyle configuration
pom.xml               # Maven project file
Jenkinsfile           # Jenkins pipeline definition
azure-pipelines.yml   # Azure DevOps pipeline (credential scanning)
README.md             # User documentation
CHANGELOG-old.md      # Historical changes
ThirdPartyNotices.txt # License notices
```

## CI/CD and Validation

### GitHub Actions Workflows

1. **Jenkins Security Scan** (`.github/workflows/jenkins-security-scan.yml`)
   - Triggers: Push to master, PRs, manual dispatch
   - Uses `jenkins-infra/jenkins-security-scan` reusable workflow
   - Requires permissions: `security-events: write`, `contents: read`, `actions: read`
   - Uses Maven cache for dependencies

2. **Continuous Delivery** (`.github/workflows/cd.yaml`)
   - Triggers: Manual dispatch, check_run completion
   - Uses `jenkins-infra/github-reusable-workflows/.github/workflows/maven-cd.yml`
   - Requires secrets: `MAVEN_USERNAME`, `MAVEN_TOKEN`

3. **Standard Jenkins CI** (Jenkinsfile)
   - Uses `buildPlugin()` from jenkins-infra/pipeline-library
   - Builds with: Linux + JDK 21, Linux + JDK 17
   - Uses container agents

### Azure Pipelines
- **azure-pipelines.yml** - Runs credential scanning (CredScan) on Windows 2019
- Not the primary CI, mainly for security validation

## Code Style Requirements

**Checkstyle is enforced during build.** Key rules:
- Max line length: 120 characters
- Line separator: LF (Unix-style)
- No trailing whitespace
- No tab characters (use spaces)
- Max file length: 2200 lines
- Max method length: default (150 lines)
- Required: proper imports (no star imports except where needed)
- Required: Javadoc style compliance
- Magic numbers trigger warnings (use constants)

**To check style before committing:**
```bash
mvn checkstyle:check
```

## Dependencies and Known Issues

### Maven Dependencies
Key dependencies (managed via BOM `bom-2.516.x`):
- `azure-credentials` - Azure authentication
- `caffeine-api` - Caching
- `commons-lang3-api` - Utilities
- `jsch` - SSH client
- `maverick-synergy-client:3.1.4` - Alternative SSH
- `configuration-as-code` - JCasC support (test scope)

### Known TODOs and Technical Debt
The codebase has several TODOs that indicate areas needing careful changes:

1. **AzureVMManagementServiceDelegate.java:**
   - SKU sizes are hardcoded, should be loaded from Azure dynamically
   - Data disk handling needs improvement

2. **AzureVMAgentSSHLauncher.java:**
   - File permission handling for scripts needs testing
   - Excluded from checkstyle compliance

3. **Retention Strategies:**
   - Methods like `check()` in retention strategy classes have TODO comments about when they're called

4. **Configuration as Code:**
   - Symbol name `azureVMCloudRetentionStrategy` may need evaluation for impact before renaming

## Testing Strategy

### Test Types
- **Unit Tests:** `src/test/java/**/*Test.java` (run with `mvn test`)
- **Integration Tests:** `src/test/java/**/IT*.java` (run with `mvn verify -Pit`)
- **JCasC Tests:** `src/test/java/.../test/jcasc/*` (validate configuration-as-code)

### Test Execution Notes
- Tests run in parallel by default
- Integration tests may require Azure credentials
- Use `-DskipTests` to skip tests during development iteration
- Always run full test suite before final submission

## Development Workflow

### Making Changes

1. **Before starting:** Understand the plugin architecture
   - Main entry: `AzureVMCloud` extends Jenkins `Cloud`
   - Templates: `AzureVMAgentTemplate` defines VM configuration
   - Provisioning: `AzureVMManagementServiceDelegate` handles Azure API calls
   - Builders: Fluent API in `builders/` package for programmatic configuration

2. **During development:**
   - Follow existing code patterns
   - Respect checkstyle rules (or add suppressions with comments)
   - Update tests if changing behavior
   - Check help files in `src/main/webapp/help-*.html` if adding UI fields

3. **Before committing:**
   - Run `mvn checkstyle:check` to verify style compliance
   - Run `mvn test` to ensure unit tests pass
   - Run `mvn package` to ensure the plugin builds
   - Verify no new checkstyle violations in files not in the exclusion list

### Common Development Tasks

**Add a new configuration field:**
1. Add field to `AzureVMAgentTemplate.java` or `AzureVMCloud.java`
2. Update corresponding builder in `builders/` package
3. Add Jelly UI file in `src/main/resources/com/microsoft/azure/vmagent/`
4. Add help HTML in `src/main/webapp/help-[fieldname].html`
5. Update tests to cover new field
6. Add to Messages.properties if adding labels

**Modify Azure integration:**
1. Changes typically go in `AzureVMManagementServiceDelegate.java`
2. Update ARM templates in `src/main/resources/*.json` if needed
3. Consider retry logic in `retry/` package
4. Update integration tests (`IT*.java`)

**Update build scripts:**
- Shell scripts in `src/main/resources/scripts/*.sh`
- Init script examples in `docs/init-scripts/`

## Important Notes

### Build Environment Limitations
- **This repository requires network access to build successfully**
- In sandboxed/offline environments, Maven will fail to resolve Jenkins parent POM and dependencies
- If you encounter network errors, document them but don't attempt to work around with offline modes
- The CI environments (GitHub Actions, Jenkins) have proper network access

### File Encoding
- All source files should use UTF-8 encoding
- Line endings must be LF (Unix-style) per checkstyle rules
- Git attributes file (`.gitattributes`) enforces consistent line endings

### Plugin Packaging
- Build output: `target/azure-vm-agents.hpi` (Jenkins plugin file)
- Version is derived from git via `git-changelist-maven-extension`
- Default version format: `9999-SNAPSHOT` (see `<changelist>` in pom.xml)

## Quick Reference

### Most Common Commands
```bash
# Validate code style
mvn checkstyle:check

# Run tests
mvn test

# Build plugin
mvn package

# Build without tests (iteration)
mvn package -DskipTests

# Full verification
mvn verify
```

### When Making Changes
1. ✅ **DO** run checkstyle before committing
2. ✅ **DO** follow existing code patterns and architecture
3. ✅ **DO** update tests when changing behavior
4. ✅ **DO** respect the 120-character line limit
5. ❌ **DON'T** add star imports
6. ❌ **DON'T** use tabs (use spaces)
7. ❌ **DON'T** introduce new checkstyle violations
8. ❌ **DON'T** modify excluded files unless absolutely necessary

### Getting Help
- **README.md** - User-facing documentation and configuration guide
- **docs/configure-with-groovy.md** - Programmatic configuration examples
- **src/main/java/com/microsoft/azure/vmagent/builders/** - Fluent builder API reference
- **Existing tests** - Best examples of how to test features

---

## Trust These Instructions

**These instructions have been validated against the actual codebase.** Only search for additional information if:
1. You need specifics about a particular class or method implementation
2. These instructions are found to be incorrect or outdated
3. You're implementing a feature not covered here

Otherwise, trust this guide to reduce exploration time and avoid common pitfalls.
