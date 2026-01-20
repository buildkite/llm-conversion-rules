# Jenkins to Buildkite Pipeline Conversion Rules

## Overview
This document provides comprehensive rules for converting Jenkins pipelines (written in Groovy DSL) to Buildkite pipelines (written in YAML). These rules are based on practical conversion experience and account for the architectural differences between the two platforms.

Note: This version of the rules will output a YAML file without asking any questions. It is intended for use on the Buildkite pipeline migration web site.

> **Note**: For general Buildkite pipeline best practices (not specific to Jenkins conversion), see [TRANSLATION_RULES_GENERAL.md](../TRANSLATION_RULES_GENERAL.md).

## Core Conversion Principles

### STAGE-TO-GROUP-MAPPING
- **Rule**: Convert each Jenkins `stage` block to a Buildkite `group` step
- **Exception**: If the Jenkins 'stage' only has one step, just translate the step and do not use a 'group' in Buildkite
- **Step Count Determination**: Count the actual executable steps (sh, bat, script blocks) within the Jenkins stage, not the containing blocks like `steps {}` or `dir {}` wrappers
- **Implementation**: 
  - Use the exact stage name as the group name
  - Create a unique `key` for each group using lowercase, dashes, and no spaces
  - Example: `"PreDeploy Validation"` → `key: "predeploy-validation"`
- **Single Step Example**:
  ```yaml
  # Jenkins: stage('Install Yetus') { steps { sh 'install command' } }
  # Buildkite: Single step, no group needed
  - label: ":gear: Install Yetus"
    command: echo "Placeholder for Yetus installation"
  ```
- **Multiple Steps Example**:
  ```yaml  
  # Jenkins: stage('Build') { steps { sh 'compile'; sh 'test'; sh 'package' } }
  # Buildkite: Multiple steps, use group
  - group: ":gear: Build"
    steps:
      - label: ":package: Compile"
        command: echo "Placeholder for compile"
      - label: ":white_check_mark: Test"  
        command: echo "Placeholder for test"
      - label: ":rocket: Package"
        command: echo "Placeholder for package"
  ```

## Input and Parameter Conversion

### JENKINS-PARAMETERS-TO-INPUT-STEPS
- **Rule**: Convert Jenkins `parameters {}` block to Buildkite `input` steps with `fields` array
- **Supported Conversions**:
  - `string` parameters → Buildkite `text` input fields
  - `text` parameters → Buildkite `text` input fields
  - `choice` parameters → Buildkite `select` input fields
  - `booleanParam` parameters → Buildkite `select` input fields with options `["true", "false"]`
- **Unsupported Parameters**:
  - `password` parameters → **DO NOT CONVERT** - Add comment recommending Buildkite secret store
  - Plugin-provided parameter types → Cannot be translated
- **Implementation**:
  ```yaml
  steps:
    # Jenkins: parameters { string(name: 'ENVIRONMENT', defaultValue: 'dev') }
    - input: "Build Parameters"
      fields:
        - text: "Environment"
          key: "environment"
          default: "dev"
        
        # Jenkins: choice(name: 'BRANCH', choices: ['main', 'develop'])
        - select: "Branch"
          key: "branch"
          options:
            - "main"
            - "develop"
        
        # Jenkins: booleanParam(name: 'DEPLOY', defaultValue: true)
        - select: "Deploy"
          key: "deploy"
          default: "true"
          options:
            - "true"
            - "false"
    
    # Jenkins: parameters { password(name: 'API_KEY') }
    # NOTE: Jenkins password parameters cannot be securely converted to Buildkite input steps.
    # Use the Buildkite secret store instead to manage sensitive values like API keys.
    # Configure secrets in your Buildkite organization settings and reference them as
    # environment variables in your pipeline steps.
  ```

### INPUT-DEPENDENCIES
- **Rule**: Use `wait` step after input steps to ensure they complete before workflow begins
- **Implementation**: Add a `- wait` step after all input steps and before the first workflow group
- **Example**:
  ```yaml
  steps:
    - input: "Build Configuration"
      fields:
        - text: "Environment"
          key: "environment"
        - select: "Stream Name"
          key: "streamname"
          options:
            - "stream1"
            - "stream2"
    
    - wait: ~  # Ensure all inputs are completed before workflow
    
    - group: ":gear: Build"
      steps:
        - label: "Start Build"
          command: echo "Starting build"
  ```

## Step Type Conversions

### ECHO-STEPS-TO-COMMENTS
- **Rule**: Convert Jenkins `echo` statements to comments in Buildkite YAML
- **Format**: `# echo: [original message]`
- **Purpose**: Preserve information while avoiding unnecessary command steps

### INPUT-STEPS-TO-BLOCK-STEPS
- **Rule**: Convert Jenkins `input` statements to Buildkite `block` steps
- **Implementation**:
  - Use exact prompt text from Jenkins
  - Add conditional logic if present in Jenkins
  - Example: `input "message"` → `block: "Confirm"` with `prompt: "message"`

### ERROR-STEPS-TO-COMMAND-STEPS
- **Rule**: Convert Jenkins `error()` calls to command steps with build cancellation
- **Implementation**:
  ```bash
  echo "ERROR: [message]"
  buildkite-agent build cancel
  exit 1
  ```

### BUILD-JOB-TO-TRIGGER-STEPS
- **Rule**: Convert Jenkins `build job` calls to Buildkite `trigger` steps
- **Process**:
  1. Use a placeholder name for the triggered job, and include a comment telling the user to update this when the job is created.
- **Implementation**:
  - Use job name as trigger target
  - Convert parameters to `env` block
  - Handle conditional logic appropriately

## Unsupported Steps

### READJSON-FUNCTIONALITY
- **Rule**: Buildkite has no native JSON parsing like Jenkins `readJSON`
- **Comment Template**:
  ```yaml
  # NOTE: Jenkins readJSON functionality conversion
  # The original Jenkins pipeline uses readJSON to parse configuration files like:
  # def inputJSON = readJSON file: "path/to/file.json"
  # 
  # Buildkite doesn't have native JSON parsing capabilities like Jenkins.
  # To implement this functionality, you have several options:
  # 1. Use shell commands with jq or similar tools in command steps
  # 2. Add parsed values to build metadata using: buildkite-agent meta-data set
  # 3. Create a custom Buildkite plugin to handle JSON configuration parsing
  # 4. Use environment variables or pipeline upload with dynamic values
  ```

### GIT-OPERATIONS-UI-CONFIGURED
- **Rule**: Jenkins SCM steps do not translate to Buildkite YAML
- **Detection**: Look for these patterns in Jenkins pipeline files:
  - `checkout scm`
  - `git url: ...`
- **Implementation**: Add comment explaining SCM configuration is in pipeline UI settings
- **Template Comment**:
  ```yaml
  # NOTE: SCM checkout conversion
  # The original Jenkins pipeline includes:
  # git branch: 'branch-name', credentialsId: 'creds', url: 'repo-url'
  # 
  # In Buildkite, SCM repository configuration is part of the pipeline settings in the UI,
  # not specified in the YAML. Configure the repository, branch, and credentials in the
  # Buildkite pipeline settings under "Repository Settings".
  ```

### PLUGINS-WITHOUT-EQUIVALENTS
- **Rule**: Document conversion approaches for unsupported Jenkins plugins
- **Common Cases**:
  - **Artifactory Plugin**: Use Maven settings.xml or environment variables
  - **JacocoPublisher**: Upload artifacts and use external processing or Buildkite plugins
  - **jUnit Plugin**: Use junit-annotate-buildkite-plugin or artifact upload
  - **Mailer Plugin**: Use Buildkite notification configurations
- **Implementation**: Provide multiple solution options with trade-offs

### CUSTOM-STEP-DEFINITIONS
- **Rule**: Jenkins custom step definitions (`def myCustomStep() { ... }`) cannot be directly translated to Buildkite
- **Implementation**: Convert to comment with refactoring guidance
- **Template**:
  ```yaml
  # NOTE: Jenkins custom step definition conversion
  # The original Jenkins pipeline includes a custom step definition:
  # def myCustomStep() {
  #   // original step implementation code here
  # }
  # 
  # Buildkite doesn't support custom step definitions in pipeline YAML.
  # To implement this functionality in Buildkite, consider these options:
  # 1. Create a custom Buildkite plugin to encapsulate this logic
  # 2. Move the logic to an external shell script and call it from command steps
  # 3. Inline the step logic directly into the command steps where it's used
  # 4. Use Buildkite's plugin ecosystem if similar functionality already exists
  ```
- **Purpose**: Preserve original logic while providing clear migration path
- **Note**: Custom step calls in the pipeline should reference the comment and suggest appropriate refactoring

### SHARED-LIBRARY-IMPORTS
- **Rule**: Jenkins shared library imports (`@Library('library-name')_`) and `load` commands cannot be directly translated to Buildkite
- **Detection**: Look for these patterns in Jenkins pipeline files:
  - `@Library('...')_` annotations at the top of pipeline files
  - `load` commands that import additional DSL code: `variableName = load 'path/to/file.groovy'`
  - Method calls on loaded variables: `variableName.methodName(parameters)`
  - Method calls that don't match known Jenkins plugins or pipeline steps: `util.avoidFaultyNodes()`
- **Implementation**: Convert to comment with manual migration guidance
- **Template**:
  ```yaml
  # NOTE: Jenkins shared library and load command conversion
  # The original Jenkins pipeline includes shared libraries and/or load commands:
  # @Library('jenkins-pipeline-shared-libraries')_
  # githubUtils = load '.ci/jenkins/shared-scripts/githubUtils.groovy'
  # githubUtils.fileIsInChangeset("${env.BRANCH_NAME}", 'incubator-kie-tools-ci-build.Dockerfile')
  # 
  # Buildkite doesn't have an equivalent to Jenkins shared libraries or load commands.
  # You'll need to manually review the shared library/loaded file code and consider these options:
  # 1. Convert shared library functions and loaded scripts to external shell scripts
  # 2. Create custom Buildkite plugins for reusable functionality
  # 3. Inline the shared library/loaded script code directly into pipeline steps where used
  # 4. Use Buildkite's plugin ecosystem if similar functionality exists
  # 
  # Review the shared library at: [library repository URL if known]
  # Review loaded files at: [path to loaded .groovy files]
  # Common shared functions may include: deployment helpers, notification utilities, 
  # environment setup, testing frameworks, custom build steps, or utility functions.
  ```
- **Purpose**: Alert users to manual work required for shared library functionality
- **Follow-up**: Any custom step calls in the pipeline that come from the shared library should include references to this conversion note

## Documentation and Comments

### PRESERVE-ORIGINAL-CONTEXT
- **Rule**: Always include comments explaining Jenkins-to-Buildkite conversions
- **Purpose**: Maintain traceability and help future developers understand changes
- **Format**: Include original Jenkins code in comments when relevant

### IMPLEMENTATION-INSTRUCTIONS
- **Rule**: Provide clear instructions for external scripts and manual steps
- **Include**:
  - File paths and naming conventions
  - Executable permissions requirements
  - Script content as comments
  - Purpose and usage explanation

## Platform-Specific Considerations

### ENVIRONMENT-VARIABLE-DIFFERENCES
- **Rule**: Map Jenkins environment variables to Buildkite equivalents
- **Research**: Check Context7 documentation for supported variables
- **Common Mappings**: Jenkins job parameters → Buildkite metadata

### PLUGIN-EXTERNAL-TOOL-HANDLING
- **Rule**: Document when Buildkite plugins might be needed
- **Recommendation**: Suggest plugin development for complex repeated patterns
- **Alternative**: Provide shell script solutions as interim approach

## Agent Execution Model Differences

### NODE-BLOCK-TO-SINGLE-STEP
- **Rule**: Everything inside a Jenkins `node` block runs on a single build agent
- **Buildkite Difference**: Every step can run on a different agent by default
- **Implementation**: Combine all commands from a Jenkins node block into a single Buildkite command step
- **Purpose**: Maintain agent consistency for file system state, git operations, and tool configurations
- **Example**:
  ```yaml
  # Jenkins: node("agent") { sh "cmd1"; sh "cmd2"; sh "cmd3" }
  # Buildkite: Single step combining all commands
  - label: "Combined Node Operations"
    agents:
      queue: "agent-queue"
    command: |
      # cmd1
      # cmd2  
      # cmd3
  ```

### PIPELINE-STRUCTURE-VARIATION
- **Rule**: Jenkins pipelines can use two different structural approaches
- **Best Practice Pattern**: `stage { node { ... } }` (stages as parents, nodes inside)
- **Alternative Pattern**: `node { stage { ... } }` (nodes as parents, stages inside)
- **Conversion Strategy**: Identify logical blocks regardless of Jenkins structure
- **Implementation**: 
  - **Best Practice**: Convert each stage to Buildkite group, combine node contents into single step
  - **Alternative**: Identify logical workflow stages within node blocks, create appropriate Buildkite groups
  - **Key Principle**: Break pipeline into logical blocks that represent distinct workflow phases

### LOGICAL-BLOCK-IDENTIFICATION
- **Rule**: Define Buildkite groups based on logical workflow phases, not just Jenkins structure
- **Analysis Process**:
  1. **Identify Workflow Phases**: Build, Test, Analysis, Deploy regardless of Jenkins structure
  2. **Group Related Operations**: Operations that logically belong together
  3. **Consider Dependencies**: What must run before what
  4. **Agent Requirements**: What needs to run on the same agent vs. different agents
- **Examples of Logical Blocks**:
  - **Provision**: Initial setup, version generation, source preparation
  - **UnitTests**: Unit test execution and result collection
  - **IntegrationTests**: Integration test execution with external services
  - **StaticAnalysis**: Code quality checks and reporting
  - **Build**: Compilation, packaging, artifact creation
  - **Deploy**: Deployment operations to various environments

## Matrix Build Pattern Detection and Conversion

### MATRIX-BUILD-PATTERN-DETECTION
- **Rule**: Detect Jenkins matrix build strategies and convert to Buildkite matrix command steps
- **Primary Patterns to Detect**:

#### Pattern 1: Axes with Combinations (jenkinsci style)
```groovy
def axes = [
  platforms: ['linux', 'windows'],
  jdks: [17, 21],
]
def builds = [:]
axes.values().combinations {
  def (platform, jdk) = it
  builds["${platform}-jdk${jdk}"] = { ... }
}
parallel builds
```

#### Pattern 2: Separate Arrays with Nested Iteration (maven-surefire style)
```groovy
final def oses = ['linux':'ubuntu']
final def mavens = ['3.x.x', '3.6.3']
final def jdks = [21, 17, 11, 8]
final Map stages = [:]
oses.eachWithIndex { osMapping, indexOfOs ->
  mavens.eachWithIndex { maven, indexOfMaven ->
    jdks.eachWithIndex { jdk, indexOfJdk ->
      stages[stageKey] = { ... }
    }
  }
}
parallel(stages)
```

### MATRIX-DETECTION-ALGORITHM
- **Rule**: Use these indicators to identify Jenkins matrix builds
- **High Confidence Indicators** (must have 3+ for matrix detection):
  - `def axes = [...]` or multiple `def/final def arrayName = [...]` definitions
  - `.values().combinations` usage OR nested `.eachWithIndex` loops
  - `def builds = [:]` or `final Map stages = [:]` initialization
  - `parallel builds` or `parallel(stages)` execution
- **Supporting Indicators**:
  - Dynamic build naming with interpolation: `"${platform}-jdk${jdk}"`
  - Variable destructuring: `def (platform, jdk) = it`
  - Conditional logic for skipping combinations: `if (...) return`
  - Agent label construction from variables: `'maven-' + jdk`

### JENKINS-TO-BUILDKITE-MATRIX
- **Rule**: Convert Jenkins matrix patterns to Buildkite `matrix.setup` configuration
- **Implementation**:

#### From Pattern 1 (axes.combinations):
```yaml
# Jenkins: def axes = [platforms: ['linux', 'windows'], jdks: [17, 21]]
steps:
  - label: ":gear: {{matrix.platform}} - JDK {{matrix.jdk}} - Build"
    command: |
      echo "Building on {{matrix.platform}} with JDK {{matrix.jdk}}"
      # Original Jenkins build commands here
    matrix:
      setup:
        platform: ["linux", "windows"]
        jdk: [17, 21]
    # Convert Jenkins conditional logic to adjustments
    adjustments:
      - with:
          platform: "windows"
          jdk: 21
        skip: true  # if (platform == 'windows' && jdk != 17) return
```

#### From Pattern 2 (nested eachWithIndex):
```yaml
# Jenkins: oses.eachWithIndex { mavens.eachWithIndex { jdks.eachWithIndex { ... }}}
steps:
  - label: ":gear: {{matrix.os}} - Maven {{matrix.maven}} - JDK {{matrix.jdk}}"
    command: |
      echo "Building on {{matrix.os}} with Maven {{matrix.maven}} and JDK {{matrix.jdk}}"
      # Original Jenkins build commands here
    matrix:
      setup:
        os: ["linux"]
        maven: ["3.x.x", "3.6.3"]
        jdk: [21, 17, 11, 8]
```

### MATRIX-VARIABLE-INTERPOLATION
- **Rule**: Convert Jenkins matrix variable access to Buildkite interpolation
- **Mappings**:
  - Jenkins: `def (platform, jdk) = it` → Buildkite: `{{matrix.platform}}`, `{{matrix.jdk}}`
  - Jenkins: `"${os}-jdk${jdk}-maven${maven}"` → Buildkite: `"{{matrix.os}}-jdk{{matrix.jdk}}-maven{{matrix.maven}}"`
  - Jenkins: `'maven-' + jdk` → Buildkite: `"maven-{{matrix.jdk}}"`
- **Agent Targeting**: Use matrix variables in agent configuration:
  ```yaml
  agents:
    queue: "{{matrix.platform}}"
    # or queue: "maven-{{matrix.jdk}}"
  ```

### MATRIX-CONDITIONAL-LOGIC
- **Rule**: Convert Jenkins matrix conditional logic to Buildkite `adjustments`
- **Common Patterns**:
  - Skip combinations: `if (condition) return` → `adjustments: - with: {...} skip: true`
  - Soft fail combinations: Add `soft_fail: true` for non-critical failures
  - Add new combinations: Use adjustments to extend beyond setup dimensions
- **Example**:
  ```yaml
  # Jenkins: if (platform == 'windows' && jdk != 17) return
  adjustments:
    - with:
        platform: "windows"
        jdk: 21
      skip: true
  ```

### MATRIX-BUILD-CONSOLIDATION
- **Rule**: Consolidate Jenkins node block contents into single Buildkite matrix step
- **Implementation**: Combine all commands from within Jenkins matrix build closures
- **Preserve Structure**: Maintain stage names and logical flow within matrix step
- **Example**:
  ```yaml
  # Combine Jenkins: stage("Checkout") + stage("Build") + stage("Publish")
  - label: ":gear: {{matrix.platform}} - JDK {{matrix.jdk}} - Full Build"
    command: |
      # Checkout (handled automatically by Buildkite)
      echo "Building on {{matrix.platform}} with JDK {{matrix.jdk}}"
      
      # Build stage commands
      ./mvnw clean install -Djava.version={{matrix.jdk}}
      
      # Publish stage commands
      buildkite-agent artifact upload "target/**/*.jar"
    matrix:
      setup:
        platform: ["linux", "windows"]
        jdk: [17, 21]
  ```

## Command Translation Rules

### WINDOWS-BATCH-COMMANDS
- **Rule**: Jenkins `bat` commands should translate to Buildkite `command` steps
- **Requirement**: Must include comment that step requires Windows agents
- **Implementation**: Add `agents.queue` targeting Windows agents
- **Comment Template**: `# NOTE: This step must run on Windows agents due to 'bat' command usage`

### JENKINS-TOOL-REFERENCES
- **Rule**: Jenkins tool path variables need manual configuration in Buildkite
- **Examples**: `${W_M2_HOME}\\bin\\mvn` → `mvn` (assuming PATH configuration)
- **Implementation**: Convert to standard command names and document tool requirements
- **Note**: Add comments about ensuring tools are available on target agents

## Artifact and State Management

### STASH-TO-ARTIFACT-UPLOAD
- **Rule**: Jenkins `stash` operations convert to Buildkite `artifact_paths`
- **Implementation**: Use `artifact_paths` array in the uploading step
- **Limitations**: Buildkite doesn't support exclude patterns like Jenkins stash
- **Workaround**: Handle exclusions in downloading steps or shell scripts

### UNSTASH-TO-ARTIFACT-DOWNLOAD
- **Rule**: Jenkins `unstash` operations convert to `buildkite-agent artifact download`
- **Implementation**: Use `buildkite-agent artifact download "pattern" . --build $BUILD_ID` in a command step
- **Runtime Variables**: Use `$$BUILDKITE_TRIGGERED_FROM_BUILD_ID` for cross-build downloads
- **Pattern**: Download artifacts from previous build steps within same pipeline

### ENVIRONMENT-VARIABLE-BLOCK
- **Rule**: Convert Jenkins `environment {}` blocks to Buildkite `env:` YAML blocks based on scope
- **Pipeline-Level Environment**: Jenkins top-level `environment {}` → Buildkite pipeline-level `env:`
- **Stage-Level Environment**: Jenkins stage-level `environment {}` → Buildkite step-level `env:` for each step in the group
- **Variable Interpolation Restriction**: Buildkite environment variable values cannot contain other variables. Patterns like `SOURCEDIR = "${WORKSPACE}/centos-7/src"` must be converted to use placeholder paths with explanatory comments
- **Implementation**:
  
  **Pipeline-Level (Jenkins top-level environment):**
  ```yaml
  # Jenkins: pipeline { environment { NAME = "myname"; VERSION = "1.0" } }
  env:
    NAME: "myname"
    VERSION: "1.0"
  
  steps:
    # All steps inherit these variables
  ```
  
  **Step-Level - Single Step (Jenkins stage environment):**
  ```yaml
  # Jenkins: stage("Build") { environment { BUILD_TYPE = "release" } }
  # Single step in stage - apply env vars directly
  steps:
    - group: ":gear: Build"
      steps:
        - label: ":package: Compile"
          command: echo "Compiling"
          env:
            BUILD_TYPE: "release"
  ```
  
  **Step-Level - Multiple Steps with YAML Anchors (Jenkins stage environment):**
  ```yaml
  # Jenkins: stage("Build") { environment { BUILD_TYPE = "release"; DEPLOY_ENV = "staging" } }
  # Multiple steps in stage - use YAML anchor for conciseness
  
  build_env: &build_env
    BUILD_TYPE: "release"
    DEPLOY_ENV: "staging"
  
  steps:
    - group: ":gear: Build"
      steps:
        - label: ":package: Compile"
          command: echo "Compiling"
          env:
            <<: *build_env
        
        - label: ":white_check_mark: Test"
          command: echo "Testing"
          env:
            <<: *build_env
        
        - label: ":rocket: Package"
          command: echo "Packaging"
          env:
            <<: *build_env
  ```
  
  **Variable Interpolation Handling (Jenkins environment with variable references):**
  ```yaml
  # Jenkins: environment { SOURCEDIR = "${WORKSPACE}/centos-7/src"; VERSION = "1.0" }
  # Problem: Buildkite doesn't support variable interpolation in env values
  
  env:
    SOURCEDIR: "/buildkite/builds/PLACEHOLDER-WORKSPACE-PATH/centos-7/src"
    VERSION: "1.0"
    # NOTE: SOURCEDIR contains a placeholder path. Replace PLACEHOLDER-WORKSPACE-PATH
    # with the actual workspace directory path used by your Buildkite agents.
    # In Buildkite, environment variable values cannot contain other variables like ${WORKSPACE}.
  ```
  
- **YAML Anchor Benefits**: When a Jenkins stage has multiple steps and environment variables, use YAML anchors (`&anchor_name` and `<<: *anchor_name`) to avoid repetition and improve maintainability
- **Anchor Naming**: Use descriptive names like `stage_env`, `build_env`, `deploy_env` based on the Jenkins stage name
- **Variable Assignment**: Also convert Jenkins `env.VAR = value` statements to Buildkite `env:` blocks
- **Scope Rule**: Jenkins stage-level environment variables must be applied to each individual step within the corresponding Buildkite group

## Error Handling and Control Flow

### TRY-CATCH-FINALLY-BLOCKS
- **Rule**: Jenkins try/catch/finally blocks don't directly translate to Buildkite
- **Buildkite Approach**: Automatic step failure handling with configurable notifications
- **Options**:
  - Use `soft_fail` for non-critical failures
  - Use `retry` configurations for transient failures
  - Use `notify` blocks for failure notifications
  - Implement error handling within shell scripts
- **Documentation**: Always document original Jenkins error handling approach

### BUILD-RESULT-MANAGEMENT
- **Rule**: Jenkins `currentBuild.result` assignments need alternative approaches
- **Buildkite Equivalents**: 
  - `currentBuild.result = 'SUCCESS'` → Step completes successfully
  - `currentBuild.result = 'FAILURE'` → `exit 1` or `buildkite-agent build cancel`
  - `currentBuild.result = 'ABORTED'` → `buildkite-agent build cancel`
- **Implementation**: Use shell script exit codes and build control commands

## Jenkins Plugin to Buildkite Annotation Conversion

### DEPRECATED-PUBLISHER-PLUGINS
- **Rule**: Convert deprecated Jenkins publisher plugins to `buildkite-agent annotate` commands
- **Purpose**: Display analysis results directly in Buildkite UI instead of relying on deprecated plugins
- **Common Plugins**:
  - `FindBugsPublisher` → Parse XML results and create info annotations
  - `CheckStylePublisher` → Count errors/warnings from XML and annotate
  - `PmdPublisher` → Count violations from XML and annotate
  - `TasksPublisher` → Scan source files for TODO/FIXME and annotate
  - `AnalysisPublisher` → Combine multiple analysis results into summary annotation

### ANNOTATION-PATTERN-ANALYSIS
- **Rule**: Use Jenkins stage names as annotation contexts for organized UI display
- **Implementation Pattern**:
  ```bash
  # Parse analysis results from generated files
  RESULT_COUNT=$(grep -c "pattern" result-file.xml || echo "0")
  
  # Create annotation using original stage name as context
  buildkite-agent annotate --style "info" --context "stage-name" \
    "**Stage Name Results**: Found $RESULT_COUNT issues. See artifacts for details."
  ```
- **Annotation Styles**: Use `info`, `warning`, `success` for visual distinction
- **Context Names**: Use original Jenkins stage names (lowercase, hyphenated)

### ANALYSIS-RESULT-PARSING
- **Rule**: Parse XML/text results to provide meaningful metrics in annotations
- **Implementation Examples**:
  - **Findbugs**: `grep -c "<BugInstance" target/findbugsXml.xml`
  - **Checkstyle**: `grep -c 'severity="error"' target/checkstyle-result.xml`
  - **PMD**: `grep -c "<violation" target/pmd.xml`
  - **TaskScanner**: `find . -name "*.java" -exec grep -c "TODO\|FIXME" {} \;`
- **Fallback**: Always provide `|| echo "0"` for missing files or zero results

### COMBINED-ANALYSIS-ANNOTATIONS
- **Rule**: Create summary annotations that combine results from multiple analysis tools
- **Purpose**: Provide overview of all static analysis results in single annotation
- **Implementation**: Use variables captured from individual tool parsing to create comprehensive summary
- **Format**: Use markdown formatting with bullet points and emojis for readability

## Maven and Build Tool Conversion

### MAVEN-GOAL-ADJUSTMENTS
- **Rule**: Some Jenkins Maven goals need adjustment for Buildkite execution
- **Examples**:
  - `checkstyle:check` → `checkstyle:checkstyle` (generate report without failing build)
  - Maven goals that fail builds should be adjusted to generate reports for annotation parsing
- **Purpose**: Allow analysis results to be captured and annotated even if quality gates fail

### MAVEN-PLUGIN-CONFIGURATION
- **Rule**: Document required Maven plugins in conversion comments
- **Implementation**: Add comprehensive notes about pom.xml requirements
- **Common Plugins**:
  - `findbugs-maven-plugin`
  - `maven-checkstyle-plugin`
  - `maven-pmd-plugin`
  - `jacoco-maven-plugin`
- **Note**: Ensure plugins are properly configured to generate expected output files

## Service and Infrastructure Dependencies

### EXTERNAL-SERVICE-MANAGEMENT
- **Rule**: Services started within Jenkins node blocks need documentation of agent requirements
- **Examples**: Database services, application servers, external tools
- **Implementation**: Document service requirements and agent configuration needs
- **Template**:
  ```yaml
  # NOTE: [Service] requirements
  # Ensure [service] is installed on the [agent-queue] agents and that the 
  # jenkins user has sudo privileges to start/stop the service
  ```

### AGENT-QUEUE-TARGETING
- **Rule**: Use specific agent queues for different types of workloads
- **Pattern**:
  - Windows-specific tasks: `queue: "windows"`
- **Implementation**: Match original Jenkins node labels as closely as possible

## Process and Workflow

### INITIAL-TRANSLATION-WORKFLOW
- **Rule**: Follow this EXACT structured process for Jenkins to Buildkite conversion
- **Process**:
  1. **Break up the Jenkins pipeline into logical blocks** based on what work is being done
  2. **Convert those blocks into Buildkite YAML**, using ONLY simple placeholder commands like `echo "Placeholder for [description]"`
  3. **Use groups when you need to run multiple commands** that are related to each other. Otherwise, use isolated commands with descriptive labels and don't use a group
  4. **STOP HERE**
- **Purpose**: Ensure incremental development and user feedback before detailed implementation
- **Warning**: Do NOT implement actual Jenkins commands in the first pass - use simple placeholders only

### PLACEHOLDER-COMMAND-FORMAT
- **Rule**: Use simple, descriptive placeholder commands during initial translation
- **Format**: `echo "Placeholder for [brief description of what this step does]"`
- **Examples**:
  - `echo "Placeholder for unit tests execution"`
  - `echo "Placeholder for static code analysis"`
  - `echo "Placeholder for deployment to staging"`
- **Purpose**: Focus on structure and dependencies without implementation complexity
- **Forbidden**: Do NOT include actual Jenkins commands, complex logic, or plugin conversions in initial translation

### DOCUMENTATION-AS-CODE
- **Rule**: Include all conversion decisions and alternatives in the final YAML
- **Purpose**: Create self-documenting pipeline configurations
- **Benefit**: Easier maintenance and future modifications

## Error Handling and Troubleshooting

### COMMON-PITFALLS
- **Rule**: Watch for these common conversion errors
- **Issues**:
  - Using build metadata in conditionals
  - Complex shell scripts in single command steps
  - Incorrect variable interpolation timing
  - Missing step dependencies
  - Not combining Jenkins node block steps into single Buildkite steps
  - Forgetting to discard Jenkins parallel wrappers
  - Converting source code `unstash` operations unnecessarily
  - Not using buildkite-agent annotate for deprecated Jenkins publisher plugins
  - Failing to adjust Maven goals that would break annotation parsing
  - Not analyzing logical workflow phases regardless of Jenkins structure pattern
  - Including agent queue specifications in initial translations
  - **NEW**: Implementing actual Jenkins commands in initial translation instead of using placeholders
  - **NEW**: Forgetting to include emojis in step labels and group names
  - **NEW**: Missing Jenkins matrix build patterns and not converting them to Buildkite matrix steps

### TESTING-AND-VERIFICATION
- **Rule**: Provide testing strategy for converted pipelines
- **Recommendation**: Test with realistic data and scenarios
- **Validation**: Ensure converted pipeline maintains original functionality

### POST-BLOCK-CONVERSION
- **Rule**: Convert Jenkins `post` blocks to comments with repository hook implementation guidance
- **Problem**: Jenkins `post` blocks define cleanup or notification steps after stages/builds complete. Buildkite has no direct equivalent.
- **Strategy**: Preserve as comments and provide Buildkite hook implementation guidance
- **Implementation**:
  - Convert `post` block contents to comments in Buildkite YAML
  - Map Jenkins post conditions to appropriate Buildkite hook types:
    - `always` → `post-command` or `pre-exit` hook
    - `success`/`failure` → `post-command` hook with exit code check
    - `cleanup` → `pre-exit` hook
  - Provide specific hook file paths (`.buildkite/hooks/post-command`)
  - Include working example hook scripts

### WITHCREDENTIALS-BLOCK-CONVERSION
- **Rule**: Convert Jenkins `withCredentials` blocks to Buildkite secret retrieval commands with configuration guidance
- **Problem**: Jenkins `withCredentials()` blocks inject secrets from Jenkins credentials store. Buildkite has different secret management.
- **Strategy**: Convert to `buildkite-agent secret get` commands with comprehensive setup guidance
- **Implementation**:
  - Map Jenkins credential types to Buildkite secret retrieval:
    - `string(credentialsId, variable)` → `buildkite-agent secret get "secret-name"`
    - `usernamePassword()` → Two separate secret gets
    - `sshUserPrivateKey()` → File-based secret with proper permissions
  - Include secret configuration checklist for Buildkite setup
  - Provide alternative approaches (Vault, AWS Secrets Manager, Azure Key Vault plugins)
  - Add security best practices (cleanup, permissions, variable evaluation)
  - Document 5-phase migration strategy

---

## Summary

These rules provide a comprehensive framework for converting Jenkins pipelines to Buildkite while accounting for the architectural differences between the platforms. The key principles are:

1. **Initial Translation Workflow**: Follow structured 6-step process for incremental conversion
2. **Agent-Free First Run**: Comment out agent specifications for initial testing and validation
3. **Structural Mapping**: Convert Jenkins stages to Buildkite groups with proper dependencies
4. **Input Handling**: Transform build parameters to input steps with appropriate types
5. **Complex Logic**: Move complex scripts to external files with proper variable interpolation
6. **Conditional Logic**: Handle limitations through shell scripts and dynamic uploads
7. **Validation**: Always validate and test converted pipelines after each change
8. **Documentation**: Preserve context and provide clear implementation instructions
9. **Visual Enhancement**: Use appropriate emojis in all labels and group names for improved user experience

Following these rules ensures reliable, maintainable, and functionally equivalent Buildkite pipelines that properly handle the conversion from Jenkins DSL to Buildkite YAML through an iterative and testable approach with enhanced visual organization.
