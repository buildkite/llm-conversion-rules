# GitHub Actions to Buildkite Translation Rules

This document provides rules for AI agents to translate GitHub Actions (GHA) workflows into Buildkite pipelines.

## Terminology

| GitHub Actions | Buildkite |
|---------------|-----------|
| Workflow | Pipeline |
| Job | Step (command step) |
| Step | Commands within a step |
| Action | Plugin |
| Runner | Agent |
| `runs-on` | `agents` |

---

## Universal Rules

These apply to every translation:

1. **Remove `on:` block** - Triggers are configured in Buildkite UI, not YAML. Generate a comment documenting original triggers.
2. **Remove `permissions:` block** - Permissions are managed via agent configuration. Generate a comment documenting required permissions.
3. **Remove `actions/checkout`** - Buildkite checks out code automatically.
4. **Convert jobs to steps** - Each GHA job becomes a Buildkite step with a `key` attribute.
5. **Combine run steps** - Multiple GHA `run` steps within a job become a single Buildkite `command` array.

---

## Quick Mappings

| GHA | Buildkite |
|-----|-----------|
| `jobs.<id>` | `steps` array item with `key: "<id>"` |
| `jobs.<id>.name` | `label` |
| `jobs.<id>.runs-on` | `agents: { queue: "..." }` |
| `jobs.<id>.env` | `env` |
| `jobs.<id>.timeout-minutes` | `timeout_in_minutes` |
| `needs` | `depends_on` |
| `continue-on-error: true` | `soft_fail: true` |
| `${{ secrets.NAME }}` | `${NAME}` (configured on agent) |
| `working-directory: ./dir` | Prepend `cd dir &&` to commands |
| `actions/upload-artifact` | `artifact_paths` on the step |
| `actions/download-artifact` | `buildkite-agent artifact download` command |

---

## Core Translation Principles

These principles guide translation decisions for patterns not explicitly covered in Quick Mappings.

### Principle 1: Triggers Require External Configuration

GitHub Actions supports ~30 webhook event triggers. Buildkite natively supports only:
- `push` (branches)
- `pull_request`
- `tag` (via "Build tags" setting)
- `schedule` (cron)

**Rule:** Remove the `on:` block and generate a header comment documenting original triggers.

**For supported triggers:**

| GHA Trigger | Buildkite Configuration |
|-------------|------------------------|
| `push` | UI → Pipeline Settings → GitHub |
| `pull_request` | UI → Pipeline Settings → GitHub |
| `schedule` | UI → Pipeline Settings → Schedules |
| `workflow_dispatch` | `input` step + "New Build" button/API |
| `release` / `create` (tags) | UI → Build tags setting |

**For unsupported triggers** (`issues`, `issue_comment`, `workflow_run`, `pull_request_target`, `merge_group`, `branch_protection_rule`, etc.):

1. **Keep in GitHub Actions** - Best for GitHub-specific automation
2. **Configure webhook** - Set up endpoint to call Buildkite API
3. **Use trigger step** - Chain from another pipeline

**For `workflow_dispatch` with `commit_sha` or `ref` inputs:**

GitHub Actions' "Run workflow" UI only allows selecting a branch, not a specific commit. Many workflows add custom inputs like `commit_sha` or `ref` to work around this limitation:

```yaml
# GHA - workaround for UI limitation
workflow_dispatch:
  inputs:
    commit_sha:
      required: false
      type: string
      description: 'Specific commit to build'
```

**Buildkite does not need this.** The "New Build" UI prompts for both commit and branch directly. Remove these inputs and any associated checkout logic:

```yaml
# GHA patterns to remove entirely:
- uses: actions/checkout@v4
  with:
    ref: ${{ github.event.inputs.commit_sha || github.sha }}

# Buildkite: No equivalent needed - automatic checkout handles this
```

**For `workflow_dispatch` with other inputs** (e.g., environment selection), convert to an `input` step:

```yaml
steps:
  - input: "Deployment Options"
    key: "options"
    fields:
      - select: "Target environment"
        key: "environment"
        required: true
        options:
          - label: "Staging"
            value: "staging"
          - label: "Production"
            value: "production"
    if: build.source == "ui"  # Only prompt for manual builds

  - label: "deploy"
    depends_on: "options"
    command:
      - ENVIRONMENT=$(buildkite-agent meta-data get "environment" --default "staging")
      - echo "Deploying to $ENVIRONMENT"
```

**For path filtering** (`paths:`, `paths-ignore:`), use the monorepo-diff plugin (see Path Filtering section).

---

### Principle 2: Context Variables Map to Environment Variables or API Calls

GHA provides rich context objects (`github.*`, `runner.*`, `env.*`). Buildkite provides environment variables.

**Rule:** Replace `${{ github.X }}` syntax with `${BUILDKITE_X}` or shell commands.

**Direct mappings:**

| GHA Context | Buildkite Environment Variable |
|------------|-------------------------------|
| `github.repository` | `BUILDKITE_REPO` or `BUILDKITE_PIPELINE_SLUG` |
| `github.sha` | `BUILDKITE_COMMIT` |
| `github.ref` | `BUILDKITE_BRANCH` |
| `github.ref_name` | `BUILDKITE_BRANCH` |
| `github.actor` | `BUILDKITE_BUILD_CREATOR` |
| `github.run_id` | `BUILDKITE_BUILD_ID` |
| `github.run_number` | `BUILDKITE_BUILD_NUMBER` |
| `github.job` | `BUILDKITE_STEP_KEY` |
| `github.workflow` | `BUILDKITE_PIPELINE_SLUG` |
| `github.event.pull_request.number` | `BUILDKITE_PULL_REQUEST` |
| `github.token` / `secrets.GITHUB_TOKEN` | `${GITHUB_TOKEN}` (configure on agent) |

**For `github.event.*` webhook payload access:**

Deep event properties (`github.event.pull_request.user.login`, `github.event.sender.*`, `github.event.inputs.*`, etc.) access the webhook payload. Options:

1. **Use `gh` CLI** to fetch data: `gh pr view $BUILDKITE_PULL_REQUEST --json author`
2. **Set at trigger time** via API when creating builds
3. **Keep in GitHub Actions** if heavily dependent on event context

**For `runner.*` context:**

| GHA | Buildkite |
|-----|-----------|
| `runner.os` | `${BUILDKITE_AGENT_META_DATA_OS}` or detect: `uname -s` |
| `runner.arch` | `${BUILDKITE_AGENT_META_DATA_ARCH}` or detect: `uname -m` |

---

### Principle 3: Expression Functions Translate to Shell Commands

GHA expression functions have no direct Buildkite equivalent. Translate to shell:

| GHA Function | Shell Equivalent |
|--------------|------------------|
| `hashFiles('**/package-lock.json')` | `find . -name "package-lock.json" -exec md5sum {} \; \| md5sum \| cut -d' ' -f1` |
| `toJson(github)` | `echo "$VAR" \| jq .` |
| `fromJSON(needs.job.outputs.matrix)` | Parse with `jq` in a script |
| `contains(x, 'y')` | `[[ "$x" == *"y"* ]]` or Buildkite `=~` operator |
| `startsWith(x, 'y')` | `[[ "$x" == "y"* ]]` or Buildkite `=~ /^y/` |
| `endsWith(x, 'y')` | `[[ "$x" == *"y" ]]` or Buildkite `=~ /y$/` |
| `format('{0}-{1}', a, b)` | `echo "${a}-${b}"` |

**For conditionals with functions:**

| GHA | Buildkite |
|-----|-----------|
| `if: contains(X, 'Y')` | `if: X =~ /Y/` |
| `if: !contains(X, 'Y')` | `if: X !~ /Y/` |
| `if: startsWith(X, 'Y')` | `if: X =~ /^Y/` |
| `if: endsWith(X, 'Y')` | `if: X =~ /Y$/` |

---

### Principle 4: Step/Job Communication Uses Meta-data or Artifacts

All GHA mechanisms for passing data between steps/jobs translate to two Buildkite primitives:

1. **Meta-data** - For small string values
2. **Artifacts** - For files

**GHA patterns and their Buildkite equivalents:**

| GHA Mechanism | Buildkite Equivalent |
|---------------|---------------------|
| `$GITHUB_OUTPUT` / `>> $GITHUB_OUTPUT` | `buildkite-agent meta-data set` |
| `jobs.<id>.outputs` | `buildkite-agent meta-data set/get` |
| `needs.<job>.outputs.<name>` | `buildkite-agent meta-data get` |
| `steps.<id>.outputs.<name>` | Variable within same step, or meta-data across steps |
| `$GITHUB_STEP_SUMMARY` | `buildkite-agent annotate` |
| `$GITHUB_ENV` | `export` within step, or meta-data across steps |

**Example - Job outputs:**

```yaml
# GHA
jobs:
  setup:
    outputs:
      version: ${{ steps.get-version.outputs.version }}
    steps:
      - id: get-version
        run: echo "version=1.2.3" >> $GITHUB_OUTPUT

# Buildkite
steps:
  - label: "setup"
    key: "setup"
    command:
      - buildkite-agent meta-data set "version" "1.2.3"

  - label: "build"
    depends_on: "setup"
    command:
      - VERSION=$(buildkite-agent meta-data get "version")
```

**Example - Step summary to annotation:**

```yaml
# GHA
- run: echo "## Build Complete" >> $GITHUB_STEP_SUMMARY

# Buildkite
- command:
    - echo "## Build Complete" | buildkite-agent annotate --style "success"
```

---

### Principle 5: Workflow-Level Settings Translation

GHA supports workflow-level `env`, `defaults`, and `secrets: inherit`. Buildkite has partial equivalents.

**Workflow-level `env`:**

Buildkite supports a top-level `env` block that applies to all steps. This maps directly from GHA:

```yaml
# GHA
env:
  REGISTRY: gcr.io
  NODE_ENV: production
jobs:
  build:
    steps:
      - run: echo $REGISTRY

# Buildkite - use top-level env block
env:
  REGISTRY: "gcr.io"
  NODE_ENV: "production"

steps:
  - label: "build"
    command:
      - echo $REGISTRY
```

**Translating `.github/env` file sourcing:**

Some workflows centralize environment config in a `.github/env` file and source it with:

```yaml
# GHA pattern to recognize
- run: cat ".github/env" >> "$GITHUB_ENV"
```

Buildkite `env` blocks only accept static values, so use a dynamic pipeline upload:

```yaml
# Buildkite translation
steps:
  - label: ":gear: Load environment"
    key: "load-env"
    command: |
      {
        echo "env:"
        sed 's/=\(.*\)/: "\1"/' .github/env | sed 's/^/  /'
      } | buildkite-agent pipeline upload

  - label: ":hammer: Next step"
    depends_on: "load-env"
    command: |
      echo "$$GOLANG_VERSION"  # Use $$ for runtime interpolation
```

Key rules:
- All steps needing these variables must have `depends_on: "load-env"`
- Use `$$VAR` (not `$VAR`) to reference these variables in downstream steps

**`defaults.run` (shell and working-directory):**

Buildkite has no equivalent for workflow-level defaults. Prepend `cd` to commands:

```yaml
# GHA
defaults:
  run:
    working-directory: ./app

# Buildkite - prepend cd to commands
steps:
  - label: "build"
    command:
      - cd app && npm install
      - cd app && npm run build
```

**`secrets: inherit` in reusable workflows:**

Configure secrets as environment variables on the agent or pass explicitly when uploading shared pipelines.

---

### Principle 6: Checkout Options are Git Commands

Buildkite's automatic checkout is equivalent to `actions/checkout` with defaults. For non-default behavior, add git commands.

| Checkout Option | Buildkite Equivalent |
|-----------------|---------------------|
| `fetch-depth: 0` | `git fetch --unshallow` or agent config `BUILDKITE_GIT_FETCH_FLAGS` |
| `ref: ${{ inputs.sha }}` | `git checkout $SHA` after default checkout |
| `persist-credentials: false` | Agent-level git credential configuration |
| `submodules: true` | `git submodule update --init` or agent config |
| `lfs: true` | `git lfs pull` or agent config |

**Example:**

```yaml
steps:
  - label: "build with full history"
    command:
      - git fetch --unshallow || true
      - git describe --tags
```

---

### Principle 7: Third-Party Actions Require Case-by-Case Handling

There are thousands of GitHub Actions. Don't try to enumerate them all.

**Decision tree for third-party actions:**

1. **Check for Buildkite plugin** at [plugins.buildkite.com](https://plugins.buildkite.com)
2. **Use the underlying CLI** - Most actions wrap a CLI tool
3. **Extract the commands** - Read the action source to find what it runs
4. **Keep in GitHub Actions** - If deeply GitHub-integrated

**Common patterns:**

| Action Pattern | Buildkite Approach |
|----------------|-------------------|
| `actions/setup-*` | Assume tool is installed; optionally use Docker plugin with appropriate image |
| `docker/*` actions | Use docker plugin or run docker commands |
| `*-lint-action` | Run linter CLI directly |
| `*-test-reporter` | Use Buildkite Test Analytics or annotations |
| GitHub API actions (`github-script`, `stale`, `labeler`) | Keep in GitHub Actions |
| Security scanning (`codeql`, `scorecard`) | Keep in GitHub Actions or use alternative tools |

**For `actions/setup-*` actions:**

When translating setup actions (`actions/setup-node`, `actions/setup-python`, `actions/setup-go`, etc.):

1. **Assume the tool is already installed** on the agent
2. **Add a comment** noting the assumption and the Docker alternative
3. **Write commands** that use the tool directly without installation logic

Do NOT generate YAML that tries to install tools if they are missing. Buildkite agents should be pre-configured with required tools, or jobs should use Docker containers.

**Example - Setup action translation:**

```yaml
# GHA
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'
- run: npm ci
- run: npm test

# Buildkite
steps:
  - label: "test"
    # Assumes Node.js 20 is installed on the agent.
    # If not available, use the Docker plugin:
    #   plugins:
    #     - docker:
    #         image: "node:20"
    plugins:
      - cache:
          manifest: package-lock.json
          path: node_modules
    command:
      - npm ci
      - npm test
```

---

### Principle 8: Status Conditionals Map to Dependencies and Build State

GHA status functions (`always()`, `failure()`, `success()`, `cancelled()`) control step execution based on job status.

| GHA | Buildkite |
|-----|-----------|
| `if: success()` | Default behavior |
| `if: always()` | `depends_on: [{ step: "X", allow_failure: true }]` |
| `if: failure()` | `depends_on` with `allow_failure: true` + `if: build.state == "failing"` |
| `if: success() \|\| failure()` | `depends_on` with `allow_failure: true` (runs unless cancelled) |
| `if: cancelled()` | No direct equivalent - add warning comment |

**Example - Always run cleanup:**

```yaml
steps:
  - label: "test"
    key: "test"
    command:
      - npm test

  - label: "cleanup"
    depends_on:
      - step: "test"
        allow_failure: true
    command:
      - npm run cleanup
```

---

### Principle 9: Repository Identity Checks Are Pipeline Settings

GHA workflows sometimes check `github.repository` to prevent forks from running sensitive jobs (releases, deployments, publishing):

```yaml
# GHA
jobs:
  release:
    if: github.repository == 'helm/helm'
    steps:
      - run: make release
```

**Buildkite equivalent:** This is handled via pipeline settings, not YAML conditionals.

**Configuration:**
1. In Buildkite UI → Pipeline Settings → GitHub
2. **Uncheck** "Build pull requests from third-party forked repositories"

This ensures:
- Forks cannot trigger builds on your pipeline
- Forks don't have access to your pipeline's secrets or build infrastructure
- Only the canonical repository can run the pipeline

**Rule:** Remove `github.repository == 'x'` checks and document in the header comment that fork builds should be disabled in pipeline settings.

**Important:** There is no Buildkite YAML conditional equivalent for checking the source repository. The following do **not** work for this purpose:
- `build.pipeline.slug` - This is the pipeline's name, not the SCM repository
- `build.repository` - Not available in conditionals

**Example translation:**

```yaml
# ============================================================================
# Translated from: release.yml
# ============================================================================
#
# SECURITY NOTE: The original workflow restricted runs to 'helm/helm' repository.
# In Buildkite, configure this in Pipeline Settings → GitHub:
#   - Uncheck "Build pull requests from third-party forked repositories"
#
# ============================================================================

steps:
  - label: ":rocket: Release"
    command:
      - make release
```

---

### Principle 10: Document Agent Infrastructure Requirements

GHA workflows specify compute requirements via `runs-on`. Buildkite uses `agents` blocks to target specific agent queues. Since agent infrastructure varies by organization, translations should document requirements rather than hardcode queue names.

**Rule:** Extract all `runs-on` values from the source workflow and generate a header comment that:
1. Lists every job and its original `runs-on` value
2. Notes any required tools (detected from setup actions or commands)
3. Provides an example `agents` block for users to customize

**Example - Agent documentation header:**

```yaml
# ============================================================================
# AGENT CONFIGURATION REQUIRED
# ============================================================================
# The original workflow used the following GitHub Actions runners:
#
#   Job                  | runs-on
#   ---------------------|----------------
#   build                | ubuntu-latest
#   test                 | ubuntu-latest
#   deploy               | ubuntu-22.04
#   macos-build          | macos-latest
#
# You must configure Buildkite agents to handle these workloads. Add an
# `agents` block to each step once your queues are set up. Example:
#
#   agents:
#     queue: "linux"
#
# Required tools on agents: Node.js 20, Docker, AWS CLI
# Alternatively, use the Docker plugin with appropriate images.
# ============================================================================
```

**For matrix builds with multiple runners:**

If a job uses `runs-on: ${{ matrix.os }}`, list all matrix values:

```yaml
#   Job                  | runs-on
#   ---------------------|----------------
#   test                 | matrix: [ubuntu-latest, macos-latest, windows-latest]
```

**For reusable workflows:**

If a job calls a reusable workflow, note that the runner is defined elsewhere:

```yaml
#   Job                  | runs-on
#   ---------------------|----------------
#   e2e-tests            | (reusable workflow - check .github/workflows/e2e.yaml)
```

---

## Detailed Translation Rules

### Job Dependencies

**GHA:**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
  test:
    needs: build
  deploy:
    needs: [build, test]
```

**Buildkite:**
```yaml
steps:
  - label: "build"
    key: "build"
    command: [...]

  - label: "test"
    key: "test"
    depends_on: "build"
    command: [...]

  - label: "deploy"
    depends_on:
      - "build"
      - "test"
    command: [...]
```

---

### Conditionals

**Simple branch conditions:**

| GHA | Buildkite |
|-----|-----------|
| `if: github.ref == 'refs/heads/main'` | `if: build.branch == "main"` |
| `if: github.event_name == 'push'` | `if: build.source == "webhook"` |
| `if: github.event_name == 'pull_request'` | `if: build.pull_request.id != null` |
| `if: github.repository == 'org/repo'` | Pipeline settings (see Principle 9) |

---

### Artifacts

**Upload:** Replace `actions/upload-artifact` with `artifact_paths`:

```yaml
steps:
  - label: "build"
    command:
      - npm run build
    artifact_paths:
      - "dist/**/*"
```

**Download:** Replace `actions/download-artifact` with `buildkite-agent artifact download`:

```yaml
steps:
  - label: "deploy"
    command:
      - buildkite-agent artifact download "dist/**/*" ./downloaded
      - npm run deploy
```

---

### Caching

Replace `actions/cache` with the cache plugin:

```yaml
steps:
  - label: "build"
    plugins:
      - cache:
          manifest: package-lock.json
          path: node_modules
          restore: pipeline
          save: pipeline
    command:
      - npm install
```

**Common configurations:**

| Language | Manifest | Path |
|----------|----------|------|
| Node.js | `package-lock.json` | `node_modules` |
| Ruby | `Gemfile.lock` | `vendor/bundle` |
| Python | `requirements.txt` | `.venv` |
| Go | `go.sum` | `~/go/pkg/mod` |

**For `cache-hit` conditional pattern:**

```yaml
# GHA - skip install if cache hit
- uses: actions/cache@v4
  id: cache
  with:
    path: node_modules
    key: nm-${{ hashFiles('package-lock.json') }}
- run: npm ci
  if: steps.cache.outputs.cache-hit != 'true'

# Buildkite - check directory exists
steps:
  - label: "build"
    plugins:
      - cache:
          manifest: package-lock.json
          path: node_modules
    command:
      - "[ -d node_modules ] || npm ci"
```

---

### Services (Sidecar Containers)

Convert GHA `services` to Docker Compose plugin:

```yaml
# docker-compose.ci.yml
version: '3.8'
services:
  app:
    build: .
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://postgres:postgres@postgres:5432/test
  
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: postgres
    healthcheck:
      test: ["CMD", "pg_isready"]
      interval: 10s
      timeout: 5s
      retries: 5
```

**Buildkite pipeline:**
```yaml
steps:
  - label: "test"
    plugins:
      - docker-compose:
          run: app
          config: docker-compose.ci.yml
    command:
      - npm test
```

---

### Container Jobs

Convert GHA `container` block to Docker plugin:

**GHA:**
```yaml
jobs:
  test:
    container:
      image: node:20
      env:
        NODE_ENV: test
      options: --user root
```

**Buildkite:**
```yaml
steps:
  - label: "test"
    plugins:
      - docker:
          image: "node:20"
          environment:
            - NODE_ENV=test
          user: root
          propagate-environment: true
    command:
      - npm test
```

---

### Matrix Builds

Buildkite has native matrix support that maps directly to GHA's `strategy.matrix`.

**Basic mapping:**

| GHA | Buildkite |
|-----|-----------|
| `strategy.matrix` | `matrix.setup` |
| `strategy.matrix.include` | `matrix.adjustments` (add combinations) |
| `strategy.matrix.exclude` | `matrix.adjustments` with `skip: true` |
| `${{ matrix.<name> }}` | `{{matrix.<name>}}` |
| `continue-on-error` per matrix combo | `soft_fail` in `adjustments` |
| `fail-fast: false` | Default behavior (siblings aren't cancelled) |
| `fail-fast: true` | No direct equivalent - use plugin or custom logic |

**Simple matrix (single dimension):**

```yaml
# GHA
strategy:
  matrix:
    node: [18, 20, 22]
steps:
  - run: node --version

# Buildkite
steps:
  - label: "test node-{{matrix}}"
    command: node --version
    matrix:
      - "18"
      - "20"
      - "22"
```

**Multi-dimensional matrix:**

```yaml
# GHA
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest]
    node: [18, 20]

# Buildkite
steps:
  - label: "test {{matrix.os}} node-{{matrix.node}}"
    command: node --version
    agents:
      queue: "{{matrix.os}}"
    matrix:
      setup:
        os:
          - "linux"
          - "macos"
        node:
          - "18"
          - "20"
```

**Excluding combinations (`exclude`):**

```yaml
# GHA
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest]
    node: [18, 20]
    exclude:
      - os: macos-latest
        node: 18

# Buildkite
steps:
  - label: "test {{matrix.os}} node-{{matrix.node}}"
    command: node --version
    matrix:
      setup:
        os:
          - "linux"
          - "macos"
        node:
          - "18"
          - "20"
      adjustments:
        - with:
            os: "macos"
            node: "18"
          skip: true
```

**Adding combinations (`include`):**

```yaml
# GHA
strategy:
  matrix:
    os: [ubuntu-latest]
    node: [18, 20]
    include:
      - os: windows-latest
        node: 20

# Buildkite
steps:
  - label: "test {{matrix.os}} node-{{matrix.node}}"
    command: node --version
    matrix:
      setup:
        os:
          - "linux"
        node:
          - "18"
          - "20"
      adjustments:
        - with:
            os: "windows"
            node: "20"
```

**Soft fail for specific combinations:**

```yaml
# GHA
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
jobs:
  - run: npm test
    continue-on-error: ${{ matrix.os == 'windows-latest' }}

# Buildkite
steps:
  - label: "test {{matrix.os}}"
    command: npm test
    matrix:
      setup:
        os:
          - "linux"
          - "windows"
      adjustments:
        - with:
            os: "windows"
          soft_fail: true
```

**Matrix limits:** Buildkite matrices support up to 6 dimensions, 20 elements per dimension, 12 adjustments, and 50 total jobs per matrix step.

**Dynamic matrix from job output (requires dynamic pipeline):**

When matrix values are computed at runtime from a previous job, use dynamic pipeline upload:

```yaml
# GHA - matrix from previous job
jobs:
  setup:
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - id: set-matrix
        run: echo "matrix={\"version\":[\"1.0\",\"2.0\"]}" >> $GITHUB_OUTPUT
  build:
    needs: setup
    strategy:
      matrix: ${{ fromJSON(needs.setup.outputs.matrix) }}

# Buildkite - dynamic pipeline upload
steps:
  - label: "setup"
    key: "setup"
    command: |
      VERSIONS=$(get-versions-somehow)
      for v in $VERSIONS; do
        cat << EOF
      - label: "build $v"
        env:
          VERSION: "$v"
        command:
          - ./build.sh
      EOF
      done | buildkite-agent pipeline upload
```

---

### Path Filtering (Monorepo)

Convert GHA `paths` / `paths-ignore` or `dorny/paths-filter` to Buildkite's native `if_changed` attribute.

**Basic usage** - run a step only when specific paths change:

```yaml
steps:
  - label: ":typescript: Build Frontend"
    command: "npm run build"
    if_changed:
      - "frontend/**"

  - trigger: "backend-deploy"
    label: ":rocket: Deploy Backend"
    if_changed:
      - "src/backend/**"
```

The `if_changed` attribute works on command steps, trigger steps, and group steps. The Buildkite agent evaluates which files changed and skips steps where no patterns match.

**Converting `dorny/paths-filter`** - GHA workflows often use a dedicated job to detect changes:

```yaml
# GHA pattern with paths-filter
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      backend: ${{ steps.filter.outputs.backend }}
    steps:
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            backend:
              - 'src/backend/**'

  deploy-backend:
    needs: changes
    if: needs.changes.outputs.backend == 'true'
    uses: ./.github/workflows/deploy-backend.yml
```

```yaml
# Buildkite equivalent - declarative condition on the step itself
steps:
  - trigger: deploy-backend
    label: ":rocket: Deploy Backend"
    if_changed:
      - "src/backend/**"
```

**Exclusion patterns** - use `include` and `exclude` for more control:

```yaml
steps:
  - label: ":test_tube: Run Tests"
    command: "make test"
    if_changed:
      include:
        - "**/*"
      exclude:
        - "**/*.md"
        - "docs/**"
```

**GHA negation patterns are not supported** - GHA allows `!pattern` to negate previous ignores:

```yaml
# GHA paths-ignore with negation
paths-ignore:
  - 'images/**'
  - '!images/nginx/**'  # Exception: DO trigger for nginx images
```

Buildkite's `if_changed` does not support pattern negation. When you encounter this pattern, add a comment in the translated pipeline noting the limitation:

```yaml
steps:
  - label: ":hammer: Build"
    command: "make build"
    if_changed:
      include:
        - "**/*"
      exclude:
        - "docs/**"
        - "deploy/**"
        - "**.md"
        - "images/**"
    # NOTE: GHA workflow used '!images/nginx/**' to exclude nginx from the ignore list.
    # Buildkite if_changed does not support pattern negation. This step will NOT trigger
    # for changes to images/nginx/** even though the original workflow intended it to.
```

---

### Concurrency (Cancel In-Progress)

GHA `concurrency.cancel-in-progress` cancels running builds when a new one starts on the same branch. Buildkite has this as a **pipeline setting**, not a step-level config:

1. Navigate to pipeline **Settings** → **Builds**
2. Enable **Cancel Intermediate Builds** (cancels running builds when new commits push)
3. Optionally enable **Skip Intermediate Builds** (skips queued builds)

Document this in the pipeline header comment:

```yaml
# Concurrency: Enable "Cancel Intermediate Builds" in pipeline settings
# to replicate GHA's cancel-in-progress behavior
```

**Note:** This is a pipeline-wide setting, not per-step. If you need more granular control, use the [concurrency gates](/docs/pipelines/configure/workflows/controlling-concurrency) feature with `concurrency` and `concurrency_group` attributes.

---

### Reusable Workflows

GHA reusable workflows (`on: workflow_call`) translate to Buildkite using one of three approaches:

| Approach | Use When |
|----------|----------|
| **Trigger steps** | Default choice - fire-and-forget, no outputs needed |
| **Dynamic pipeline upload** | Need outputs, shared secrets, or same-build context |
| **YAML anchors** | Simple command reuse within a single file |

---

#### Approach 1: Trigger Steps (Preferred)

Invoke a separate pipeline. Simple and works for most reusable workflow patterns.

```yaml
steps:
  - trigger: "build-image"
    label: ":docker: Build Controller"
    build:
      env:
        INPUT_NAME: "controller"
```

Pass inputs via `build.env`. Combine with `if_changed` for path filtering:

```yaml
steps:
  - trigger: "build-image"
    label: ":docker: Build Controller"
    if_changed:
      - "cmd/nginx/**"
    build:
      env:
        INPUT_NAME: "controller"
```

**Limitation:** Triggered builds cannot return outputs to the caller. If the reusable workflow uses `outputs:`, use dynamic pipeline upload instead.

---

#### Approach 2: Dynamic Pipeline Upload

Use when the reusable workflow returns outputs or needs shared build context (secrets, meta-data, concurrency).

**Shared workflow file:**

```yaml
# .buildkite/shared/build-image.yml
steps:
  - label: ":docker: Build ${INPUT_NAME}"
    key: "build-${INPUT_NAME}"
    command:
      - IMAGE_TAG=$(./scripts/build-image.sh "${INPUT_NAME}")
      - buildkite-agent meta-data set "image_tag_${INPUT_NAME}" "$IMAGE_TAG"
```

**Caller pipeline:**

```yaml
steps:
  - label: ":gear: Build Controller"
    env:
      INPUT_NAME: "controller"
    command:
      - buildkite-agent pipeline upload .buildkite/shared/build-image.yml

  - label: ":ship: Deploy"
    depends_on: "build-controller"
    command:
      - CONTROLLER_TAG=$(buildkite-agent meta-data get "image_tag_controller")
      - ./deploy.sh --tag "$CONTROLLER_TAG"
```

**Mapping:** GHA `with:` → Buildkite `env:`, GHA `outputs:` → `buildkite-agent meta-data set/get`

---

#### Approach 3: YAML Anchors

For command reuse within a single file:

```yaml
definitions:
  deploy_commands: &deploy_commands
    - ./deploy.sh --env "${ENVIRONMENT}"

steps:
  - label: ":rocket: Deploy Staging"
    env:
      ENVIRONMENT: "staging"
    command: *deploy_commands

  - label: ":rocket: Deploy Production"
    env:
      ENVIRONMENT: "production"
    command: *deploy_commands
```

**Limitation:** Single file only - cannot share across pipelines.

---

### Environment Protection

Convert GHA `environment` with protection rules to block steps:

```yaml
steps:
  - block: ":rocket: Deploy to production?"
    prompt: "This will deploy to production. Continue?"
    key: "approve-production"

  - label: "deploy to production"
    depends_on: "approve-production"
    command:
      - npm run deploy
```

**Limitations:** Buildkite block steps don't support user-specific approval like GHA required reviewers.

---

### Retry on Failure

GHA doesn't have built-in retry. If the workflow uses retry logic, convert to Buildkite's native retry:

```yaml
steps:
  - label: "flaky-test"
    retry:
      automatic:
        - exit_status: "*"
          limit: 2
    command:
      - npm test
```

---

## Permissions

**Rule:** Remove the `permissions:` block and generate a header comment documenting required permissions.

| GHA Permission | Buildkite Equivalent |
|----------------|---------------------|
| `contents: read` | Automatic (Git checkout) |
| `contents: write` | GitHub token with `repo` scope |
| `packages: read/write` | Registry credentials on agent |
| `issues: write` | GitHub token with `issues` scope |
| `pull-requests: write` | GitHub token with `pull_requests` scope |
| `id-token: write` | Buildkite OIDC or agent IAM roles |
| `statuses: write` | Automatic via Buildkite GitHub integration |
| `checks: write` | GitHub token or Buildkite Test Analytics |
| `repository-projects: write` | GitHub token with appropriate scope |

---

## Plugin Reference

| Purpose | Plugin |
|---------|--------|
| Caching | `cache` |
| Docker containers | `docker` |
| Docker Compose | `docker-compose` |

---

## Translation Checklist

- [ ] Remove `on:` triggers and generate documentation comment
- [ ] Remove `permissions:` and generate documentation comment
- [ ] Document all `runs-on` values in header with agent configuration guidance (Principle 10)
- [ ] Remove `actions/checkout` steps
- [ ] Convert jobs to steps with `key` attributes
- [ ] Convert `needs` to `depends_on`
- [ ] Replace GitHub context variables with Buildkite equivalents
- [ ] Replace `actions/cache` with cache plugin
- [ ] Convert `services` to docker-compose plugin
- [ ] Use native Buildkite matrix support; reserve dynamic pipeline for runtime-computed matrices
- [ ] Convert workflow-level `env` to top-level `env` block; handle `defaults` per-step
- [ ] Translate expression functions to shell commands
- [ ] Flag GitHub-specific actions for manual handling
- [ ] Flag unsupported triggers for manual handling
