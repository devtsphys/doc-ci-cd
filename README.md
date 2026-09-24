# CI/CD Tools Reference Card
### GitHub Actions · GitLab CI/CD · Jenkins · Azure Pipelines · CircleCI
*Comprehensive quick-reference for syntax, structure, and common patterns.*

---

## Table of Contents

1. [Core Concepts & Terminology Map](#1-core-concepts--terminology-map)
2. [GitHub Actions](#2-github-actions)
3. [GitLab CI/CD](#3-gitlab-cicd)
4. [Jenkins](#4-jenkins)
5. [Azure Pipelines](#5-azure-pipelines)
6. [CircleCI](#6-circleci)
7. [Cross-Platform Comparison Table](#7-cross-platform-comparison-table)
8. [Common Patterns, Side-by-Side](#8-common-patterns-side-by-side)
9. [CLI Cheat Sheet](#9-cli-cheat-sheet)
10. [Cron / Schedule Syntax](#10-cron--schedule-syntax)
11. [Security Best Practices Checklist](#11-security-best-practices-checklist)
12. [Glossary](#12-glossary)

---

## 1. Core Concepts & Terminology Map

Every CI/CD platform models the same idea (code change → automated steps → feedback/deployment) with different vocabulary:

| Generic Concept | GitHub Actions | GitLab CI/CD | Jenkins | Azure Pipelines | CircleCI |
|---|---|---|---|---|---|
| Config file(s) | `.github/workflows/*.yml` | `.gitlab-ci.yml` | `Jenkinsfile` | `azure-pipelines.yml` | `.circleci/config.yml` |
| Config language | YAML | YAML | Groovy DSL | YAML | YAML |
| Top-level run | **Workflow** | **Pipeline** | **Pipeline** (Build) | **Pipeline** (Run) | **Workflow** |
| Group of steps | **Job** | **Job** | **Stage** | **Job** | **Job** |
| Logical phase | (via `needs`/ordering) | **Stage** | **Stage** | **Stage** | (via workflow graph) |
| Atomic unit | **Step** | script line(s) | **Step** | **Task**/Step | **Step** |
| Execution host | **Runner** | **Runner** | **Agent**/Node | **Agent** (in a Pool) | **Executor** |
| Trigger config | `on:` | `rules:` / `workflow:` | `triggers {}` | `trigger:` / `pr:` | workflow `triggers`/filters |
| Reusable units | Reusable workflow, Composite/Custom Action | `include:` + `extends:`, Components | Shared Library | Templates | Orbs |
| Parallel/matrix | `strategy.matrix` | `parallel:` / `parallel:matrix` | `parallel {}` (Declarative) | `strategy.matrix` | `matrix` param in workflow |
| Secrets store | Repo/Org/Env **Secrets** | CI/CD **Variables** (masked/protected) | **Credentials** store | **Variable Groups** / Key Vault | **Contexts** / Project env vars |
| Caching | `actions/cache` | `cache:` keyword | Plugin (e.g., `job-cache`) or manual | `Cache@2` task | `save_cache` / `restore_cache` |
| Artifacts | `actions/upload-artifact` | `artifacts:` keyword | `archiveArtifacts` | `PublishBuildArtifacts@1` / Pipeline artifacts | `store_artifacts` |
| Manual gate | `environment` protection rules | `when: manual` | `input` step | Approvals on **Environments** | `type: approval` job |
| Marketplace | GitHub Marketplace (Actions) | (Templates/Components) | Plugins | Marketplace (Tasks/Extensions) | Orb Registry |

---

## 2. GitHub Actions

### 2.1 File Anatomy

- Location: `.github/workflows/<name>.yml` (one or more files; each is an independent **workflow**)
- A repo can have many workflow files, each triggered independently

```yaml
name: CI                       # Display name in the Actions tab

on:                             # Trigger(s)
  push:
    branches: [main, "release/**"]
  pull_request:
    branches: [main]
  workflow_dispatch:            # Manual "Run workflow" button
    inputs:
      environment:
        description: "Target environment"
        required: true
        default: "staging"
        type: choice
        options: [staging, production]
  schedule:
    - cron: "0 3 * * *"         # 03:00 UTC daily

permissions:                    # GITHUB_TOKEN scopes (principle of least privilege)
  contents: read
  id-token: write               # needed for OIDC cloud login

env:                             # Global env vars, available to all jobs
  NODE_ENV: production

concurrency:                     # Cancel superseded runs on the same ref
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest        # or windows-latest, macos-latest, self-hosted, [self-hosted, linux, gpu]
    timeout-minutes: 15
    strategy:
      fail-fast: false
      matrix:
        node-version: [18, 20, 22]
        os: [ubuntu-latest, windows-latest]
    outputs:
      artifact-name: ${{ steps.build.outputs.name }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run build
        id: build
        run: |
          npm run build
          echo "name=dist-${{ matrix.node-version }}" >> "$GITHUB_OUTPUT"

      - name: Run tests
        run: npm test
        env:
          CI: true

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: dist-${{ matrix.node-version }}
          path: dist/
          retention-days: 7

  deploy:
    needs: build                  # Waits for 'build' job(s) to succeed
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:                  # Protection rules + required reviewers configured in repo settings
      name: production
      url: https://example.com
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: dist-20

      - name: Deploy
        run: ./deploy.sh
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

### 2.2 Trigger Types (`on:`)

| Trigger | Fires on |
|---|---|
| `push` | Commits pushed to matching branches/tags |
| `pull_request` / `pull_request_target` | PR opened/synced against base repo |
| `workflow_dispatch` | Manual trigger via UI/API/CLI, supports `inputs:` |
| `schedule` | Cron (UTC only) |
| `workflow_call` | Makes this workflow **reusable**, callable from other workflows |
| `release` | Release published/created/edited |
| `repository_dispatch` | Custom external webhook trigger |
| `issues`, `issue_comment`, `label`, etc. | GitHub event webhooks (dozens available) |
| `workflow_run` | Triggered when another workflow completes |

### 2.3 Contexts & Expressions

```yaml
${{ github.sha }}              # commit SHA
${{ github.ref }}              # refs/heads/main
${{ github.ref_name }}         # main
${{ github.event_name }}       # push, pull_request, ...
${{ github.actor }}            # user who triggered the run
${{ github.repository }}       # owner/repo
${{ runner.os }}                # Linux/Windows/macOS
${{ job.status }}               # success/failure/cancelled
${{ needs.build.outputs.x }}    # output from a dependency job
${{ matrix.node-version }}      # current matrix value
${{ secrets.MY_SECRET }}
${{ vars.MY_VARIABLE }}         # repo/org "Variables" (non-secret)
${{ steps.<id>.outputs.<name> }}
```

Common functions: `contains()`, `startsWith()`, `endsWith()`, `format()`, `join()`, `toJSON()`, `fromJSON()`, `always()`, `success()`, `failure()`, `cancelled()`.

```yaml
- name: Only on failure
  if: failure()
  run: echo "notify team"

- name: Only on main branch pushes
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

### 2.4 Secrets & Variables

- **Repository / Organization / Environment secrets**: Settings → Secrets and variables → Actions
- Accessed as `${{ secrets.NAME }}`; never printed in logs (auto-masked)
- Non-secret config: `${{ vars.NAME }}`
- **OIDC** (recommended over long-lived cloud keys):

```yaml
permissions:
  id-token: write
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/gh-actions-role
      aws-region: us-east-1
```

### 2.5 Caching & Artifacts

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.npm
      node_modules
    key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      npm-${{ runner.os }}-
```

Artifacts persist build outputs beyond the job's runner and between jobs; caches speed up dependency installs across runs. Artifacts are downloadable from the UI; caches are not.

### 2.6 Reusable Workflows & Composite Actions

**Reusable workflow** (called with `uses:` at job level):
```yaml
# .github/workflows/reusable-build.yml
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: "20"
    secrets:
      NPM_TOKEN:
        required: true
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "building with node ${{ inputs.node-version }}"
```
```yaml
# caller workflow
jobs:
  call-build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      node-version: "22"
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

**Composite action** (bundles multiple steps into one reusable step):
```yaml
# .github/actions/setup-project/action.yml
name: "Setup Project"
inputs:
  node-version:
    required: false
    default: "20"
runs:
  using: "composite"
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
    - run: npm ci
      shell: bash
```

### 2.7 CLI (`gh`) Cheat Sheet

```bash
gh workflow list                          # list workflows
gh workflow run ci.yml -f environment=staging   # manual dispatch with inputs
gh run list --workflow=ci.yml
gh run view <run-id> --log
gh run watch <run-id>
gh run rerun <run-id> --failed
gh run cancel <run-id>
gh secret set DEPLOY_TOKEN --body "xxxx"
gh api repos/:owner/:repo/actions/runs      # raw REST access
act -j build                              # run workflows locally (3rd-party tool)
```

---

## 3. GitLab CI/CD

### 3.1 File Anatomy

- Location: `.gitlab-ci.yml` at repo root (can `include:` other files)
- Executed by GitLab **Runners** (shared, group, or project-specific)

```yaml
stages:
  - build
  - test
  - deploy

default:                          # applies to all jobs unless overridden
  image: node:20
  tags: [docker]
  before_script:
    - npm ci

variables:
  NODE_ENV: "production"

workflow:                         # controls whether a pipeline runs at all
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

build-job:
  stage: build
  script:
    - npm run build
  artifacts:
    paths: [dist/]
    expire_in: 1 week

test-job:
  stage: test
  script:
    - npm test
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
  parallel:
    matrix:
      - NODE_VERSION: ["18", "20", "22"]

deploy-staging:
  stage: deploy
  script:
    - ./deploy.sh staging
  environment:
    name: staging
    url: https://staging.example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy-prod:
  stage: deploy
  script:
    - ./deploy.sh production
  environment:
    name: production
    url: https://example.com
  when: manual                    # requires clicking "play" in the UI
  needs: ["deploy-staging"]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

### 3.2 Job Keywords Reference

| Keyword | Purpose |
|---|---|
| `stage` | Which stage this job belongs to (stages run sequentially; jobs within a stage run in parallel) |
| `script` | Required — shell commands to run |
| `before_script` / `after_script` | Run before/after `script`, per-job or via `default:` |
| `image` | Docker image to run the job in |
| `services` | Sidecar containers (e.g., `postgres:15`, `docker:dind`) |
| `tags` | Selects which runners can pick up the job |
| `rules` | Modern conditional logic (replaces `only`/`except`) |
| `only` / `except` | Legacy conditional logic |
| `needs` | Defines a DAG — run as soon as dependencies finish, ignoring stage order |
| `artifacts` | Files to pass to later stages / download from UI |
| `cache` | Files to persist between pipeline runs (per branch/key) |
| `environment` | Marks a deployment target, enables environment tracking & rollback |
| `when` | `on_success` (default), `on_failure`, `always`, `manual`, `delayed` |
| `extends` | Inherit from a reusable/hidden job template (`.template_name`) |
| `parallel` / `parallel:matrix` | Run N copies of a job, optionally with different variable combos |
| `retry` | Auto-retry on failure (`max`, `when`) |
| `interruptible` | Allow the job to be cancelled by a newer pipeline |
| `resource_group` | Enforce mutual exclusion (e.g., only one deploy at a time) |
| `trigger` | Starts a child/downstream pipeline |

### 3.3 Rules Examples

```yaml
job:
  script: ./test.sh
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: on_success
    - if: '$CI_MERGE_REQUEST_ID'
      changes: ["src/**/*"]
    - when: never
```

### 3.4 Predefined Variables (selection)

```
$CI_COMMIT_SHA / $CI_COMMIT_SHORT_SHA
$CI_COMMIT_BRANCH / $CI_COMMIT_TAG
$CI_PIPELINE_ID / $CI_JOB_ID
$CI_PROJECT_NAME / $CI_PROJECT_PATH
$CI_MERGE_REQUEST_IID
$CI_PIPELINE_SOURCE   # push, merge_request_event, schedule, web, trigger, api
$CI_REGISTRY / $CI_REGISTRY_IMAGE   # built-in container registry
$CI_ENVIRONMENT_NAME
```
CI/CD variables (custom secrets) are set in **Settings → CI/CD → Variables**, with `Protect variable` and `Mask variable` flags.

### 3.5 Caching vs Artifacts

```yaml
cache:
  key:
    files: [package-lock.json]
  paths:
    - node_modules/
  policy: pull-push   # or 'pull' on jobs that shouldn't re-upload
```
- **Cache**: speeds up jobs across pipelines, best-effort, not guaranteed to exist
- **Artifacts**: guaranteed output of a specific pipeline run, browsable in UI, passed between stages

### 3.6 `extends` & `include` (DRY configs)

```yaml
.deploy_template:
  image: alpine
  before_script:
    - apk add curl

deploy-staging:
  extends: .deploy_template
  script: ./deploy.sh staging

include:
  - local: ".gitlab/ci/build.yml"
  - project: "my-group/ci-templates"
    ref: main
    file: "/templates/security-scan.yml"
  - template: Security/SAST.gitlab-ci.yml   # GitLab-managed template
```

### 3.7 Child Pipelines

```yaml
trigger-child:
  stage: deploy
  trigger:
    include: child-pipeline.yml
    strategy: depend   # parent waits for child result
```

### 3.8 CLI (`glab`) Cheat Sheet

```bash
glab ci status                     # pipeline status for current branch
glab ci view                       # interactive pipeline viewer
glab ci trace <job-id>              # tail logs of a job
glab ci run                        # manually trigger a pipeline
glab ci retry <job-id>
glab ci lint .gitlab-ci.yml         # validate config syntax
gitlab-runner exec docker test-job  # run a job locally (needs local runner install)
```

---

## 4. Jenkins

### 4.1 Jenkinsfile: Declarative vs Scripted

Jenkins pipelines are written in **Groovy**. **Declarative** syntax (structured, recommended) is far more common than **Scripted** (full Groovy flexibility).

### 4.2 Declarative Pipeline — Full Example

```groovy
pipeline {
    agent any                      // or: agent { docker { image 'node:20' } }
                                    // or: agent { label 'linux && docker' }

    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['staging', 'production'], description: 'Deploy target')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run test suite')
    }

    triggers {
        cron('H 2 * * *')           // 'H' = Jenkins-hashed minute, spreads load
        pollSCM('H/5 * * * *')
        // githubPush()             // requires GitHub plugin webhook
    }

    environment {
        NODE_ENV = 'production'
        DEPLOY_TOKEN = credentials('deploy-token-id')   // pulled from Credentials store
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'npm ci'
                sh 'npm run build'
            }
        }

        stage('Test') {
            when { expression { params.RUN_TESTS } }
            steps {
                sh 'npm test -- --reporters=junit'
            }
            post {
                always {
                    junit 'reports/junit.xml'
                }
            }
        }

        stage('Parallel Checks') {
            parallel {
                stage('Lint') {
                    steps { sh 'npm run lint' }
                }
                stage('Security Scan') {
                    steps { sh 'npm audit' }
                }
            }
        }

        stage('Approval') {
            when { branch 'main' }
            steps {
                input message: "Deploy to ${params.ENVIRONMENT}?", ok: 'Deploy'
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh "./deploy.sh ${params.ENVIRONMENT}"
            }
        }
    }

    post {
        success   { echo 'Pipeline succeeded' }
        failure   { mail to: 'team@example.com', subject: "Build Failed: ${env.JOB_NAME}", body: "${env.BUILD_URL}" }
        always    { archiveArtifacts artifacts: 'dist/**', fingerprint: true }
        cleanup   { cleanWs() }
    }
}
```

### 4.3 Scripted Pipeline — Example

```groovy
node('linux') {
    stage('Checkout') {
        checkout scm
    }
    stage('Build') {
        sh 'npm ci && npm run build'
    }
    try {
        stage('Test') {
            sh 'npm test'
        }
    } catch (err) {
        currentBuild.result = 'FAILURE'
        throw err
    } finally {
        junit 'reports/junit.xml'
    }
}
```

### 4.4 Directive Reference (Declarative)

| Directive | Purpose |
|---|---|
| `agent` | Where the pipeline/stage executes (`any`, `none`, `label`, `docker`, `kubernetes`) |
| `stages` / `stage` | Sequential phases, shown in Blue Ocean/Stage View |
| `steps` | Actual commands within a stage |
| `environment` | Env vars, including `credentials()` bindings |
| `parameters` | Build parameters, prompted or passed via API |
| `triggers` | `cron`, `pollSCM`, `upstream`, webhook triggers |
| `options` | Pipeline behavior: timeout, retry, discard old builds, etc. |
| `post` | Actions on outcome: `always`, `success`, `failure`, `unstable`, `changed`, `cleanup` |
| `when` | Conditional stage execution (`branch`, `expression`, `changelog`, `changeset`, `environment`) |
| `parallel` | Run sibling stages concurrently |
| `input` | Manual approval gate, pauses pipeline |
| `matrix` | Declarative matrix builds across axes |

### 4.5 Common Steps

```groovy
sh 'echo hello'                       // shell (Linux/macOS)
bat 'echo hello'                      // Windows batch
checkout scm                          // checkout configured SCM
git url: 'https://github.com/org/repo.git', branch: 'main'
archiveArtifacts artifacts: 'build/**', allowEmptyArchive: true
junit 'test-results/*.xml'
stash name: 'built', includes: 'dist/**'
unstash 'built'
withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
    sh 'docker login -u $USER -p $PASS'
}
timeout(time: 5, unit: 'MINUTES') { sh './long-task.sh' }
retry(3) { sh './flaky-step.sh' }
```

### 4.6 Shared Libraries (reuse across Jenkinsfiles)

```groovy
// Jenkinsfile
@Library('my-shared-library') _
myCustomBuildStep(env: 'staging')
```
```groovy
// vars/myCustomBuildStep.groovy in the shared-library repo
def call(Map config) {
    sh "./build.sh ${config.env}"
}
```

### 4.7 CLI & Automation

```bash
java -jar jenkins-cli.jar -s http://jenkins-host/ build my-job -f    # trigger build, wait for it
java -jar jenkins-cli.jar -s http://jenkins-host/ list-jobs
curl -X POST JENKINS_URL/job/my-job/build --user user:api_token
curl -X POST JENKINS_URL/job/my-job/buildWithParameters?ENV=staging
```
Blue Ocean provides a visual pipeline editor/viewer; `Jenkinsfile Linter` can validate syntax:
```bash
curl -X POST -u user:token -F "jenkinsfile=<Jenkinsfile" JENKINS_URL/pipeline-model-converter/validate
```

---

## 5. Azure Pipelines

### 5.1 File Anatomy

- Location: `azure-pipelines.yml` (or any path referenced in the pipeline definition)
- Hierarchy: **Pipeline → Stages → Jobs → Steps/Tasks**

```yaml
trigger:                          # CI trigger (push)
  branches:
    include: [main, release/*]
  paths:
    exclude: [docs/*]

pr:                                # PR validation trigger
  branches:
    include: [main]

pool:
  vmImage: "ubuntu-latest"         # or 'windows-latest', 'macOS-latest', or a self-hosted pool name

variables:
  buildConfiguration: "Release"
  - group: "prod-secrets"          # linked Variable Group

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - task: UseNode@3
            inputs:
              version: "20.x"
          - script: npm ci
            displayName: "Install dependencies"
          - script: npm run build
            displayName: "Build"
          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: "dist"
              ArtifactName: "drop"

  - stage: Test
    dependsOn: Build
    jobs:
      - job: TestJob
        strategy:
          matrix:
            node18:
              nodeVersion: "18.x"
            node20:
              nodeVersion: "20.x"
          maxParallel: 2
        steps:
          - task: UseNode@3
            inputs:
              version: $(nodeVersion)
          - script: npm test

  - stage: DeployProd
    dependsOn: Test
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployJob
        environment: "production"    # environment approvals/checks configured in Azure DevOps UI
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop
                - script: ./deploy.sh
                  env:
                    DEPLOY_TOKEN: $(deployToken)
```

### 5.2 Key Building Blocks

| Element | Notes |
|---|---|
| `trigger` | CI trigger (push-based); `none` disables CI trigger (e.g., for manual-only pipelines) |
| `pr` | Pull-request validation trigger |
| `schedules` | Cron-based scheduled runs |
| `resources` | External repos, pipelines, containers, packages consumed by this pipeline |
| `pool` | Agent pool (Microsoft-hosted `vmImage` or self-hosted `name`) |
| `stages` | Highest grouping, run sequentially by default (or in parallel via `dependsOn: []`) |
| `jobs` | Unit scheduled onto a single agent; `job`, `deployment`, in a stage |
| `steps` | `script`, `bash`, `pwsh`, `task` |
| `task` | Pre-built, versioned unit from the Marketplace (e.g., `Docker@2`, `AzureWebApp@1`) |
| `variables` | Inline, variable groups, or Key Vault-linked |
| `condition` | Expression controlling whether a stage/job/step runs |
| `strategy.matrix` | Run job N times with different variable sets |
| `templates` | Reusable YAML fragments (`extends:`, `- template:`) |
| `environment` | Deployment target with approvals/checks & history, used with `deployment` jobs |

### 5.3 Variables & Key Vault

```yaml
variables:
  - group: "shared-variable-group"
  - name: myVar
    value: "hello"

steps:
  - task: AzureKeyVault@2
    inputs:
      azureSubscription: "my-service-connection"
      KeyVaultName: "my-keyvault"
      SecretsFilter: "*"
```
Reference: `$(myVar)` in YAML, `$env:MYVAR` in PowerShell steps, `$MYVAR` in bash steps.

### 5.4 Templates (reuse)

```yaml
# templates/build-steps.yml
parameters:
  - name: nodeVersion
    type: string
    default: "20.x"
steps:
  - task: UseNode@3
    inputs:
      version: ${{ parameters.nodeVersion }}
  - script: npm ci

# azure-pipelines.yml
steps:
  - template: templates/build-steps.yml
    parameters:
      nodeVersion: "22.x"
```

### 5.5 CLI (`az pipelines`) Cheat Sheet

```bash
az pipelines create --name my-pipeline --repository my-repo --branch main --yml-path azure-pipelines.yml
az pipelines list
az pipelines run --name my-pipeline --branch main
az pipelines build queue --definition-id 12
az pipelines runs list --pipeline-ids 12
az pipelines runs show --id 456
az pipelines variable-group create --name prod-secrets --variables key=value
```

---

## 6. CircleCI

### 6.1 File Anatomy

- Location: `.circleci/config.yml`
- `version: 2.1` unlocks orbs, reusable commands, and parameters

```yaml
version: 2.1

orbs:
  node: circleci/node@6.2.0

executors:
  docker-executor:
    docker:
      - image: cimg/node:20.10
      - image: cimg/postgres:15.0   # sidecar service container

commands:                          # reusable step sequences
  install_and_cache:
    steps:
      - restore_cache:
          keys: [deps-v1-{{ checksum "package-lock.json" }}]
      - run: npm ci
      - save_cache:
          key: deps-v1-{{ checksum "package-lock.json" }}
          paths: [node_modules]

jobs:
  build:
    executor: docker-executor
    steps:
      - checkout
      - install_and_cache
      - run:
          name: Build
          command: npm run build
      - persist_to_workspace:
          root: .
          paths: [dist]
      - store_artifacts:
          path: dist
          destination: build-output

  test:
    executor: docker-executor
    parallelism: 4                  # splits tests across 4 containers
    steps:
      - checkout
      - install_and_cache
      - run:
          name: Run tests
          command: |
            TESTFILES=$(circleci tests glob "test/**/*.spec.js" | circleci tests split --split-by=timings)
            npx jest $TESTFILES --reporters=default --reporters=jest-junit
      - store_test_results:
          path: test-results

  deploy:
    docker:
      - image: cimg/base:2024.01
    steps:
      - attach_workspace:
          at: .
      - run:
          name: Deploy
          command: ./deploy.sh
          environment:
            DEPLOY_ENV: production

workflows:
  build-test-deploy:
    jobs:
      - build
      - test:
          requires: [build]
      - hold-for-approval:
          type: approval
          requires: [test]
          filters:
            branches:
              only: main
      - deploy:
          requires: [hold-for-approval]
          filters:
            branches:
              only: main
      - matrix-test:
          matrix:
            parameters:
              node-version: ["18.20", "20.10", "22.1"]
          requires: [build]
```

### 6.2 Key Concepts

| Element | Purpose |
|---|---|
| `orbs` | Shareable, versioned packages of jobs/commands/executors (like a library import) |
| `executors` | Reusable execution environments: `docker`, `machine`, `macos`, `windows` |
| `commands` | Reusable named sequences of steps (like a function) |
| `jobs` | Named units of work, each with their own executor & steps |
| `workflows` | Orchestrate jobs: order, `requires`, `filters`, `matrix`, approvals |
| `parallelism` | Splits one job's test run across N containers |
| `steps` | `checkout`, `run`, `save_cache`/`restore_cache`, `store_artifacts`, `store_test_results`, `persist_to_workspace`/`attach_workspace` |
| `workspace` | Passes files between jobs *within the same workflow run* (ephemeral) |
| `cache` | Persists across workflow runs (keyed, best-effort) |
| `type: approval` | Manual gate job in a workflow |

### 6.3 Workflow Filters & Scheduling

```yaml
workflows:
  nightly:
    triggers:
      - schedule:
          cron: "0 3 * * *"
          filters:
            branches:
              only: [main]
    jobs:
      - build
      - test
```

### 6.4 CLI Cheat Sheet

```bash
circleci config validate                          # lint config.yml
circleci config process .circleci/config.yml       # expand orbs/reuse for debugging
circleci local execute --job build                 # run a job locally (Docker only, no workflows)
circleci orb list                                  # browse orbs
circleci orb info circleci/node
circleci tests glob "test/**/*.spec.js"             # helper for test splitting
```

---

## 7. Cross-Platform Comparison Table

| Feature | GitHub Actions | GitLab CI/CD | Jenkins | Azure Pipelines | CircleCI |
|---|---|---|---|---|---|
| Hosting model | SaaS (+ self-hosted runners) | SaaS/self-managed (+ runners) | Self-hosted (mostly) | SaaS (+ self-hosted agents) | SaaS (+ self-hosted runners) |
| Config format | YAML | YAML | Groovy | YAML | YAML |
| Native Docker support | Yes (`container:`, service containers) | Yes (`image:`, `services:`) | Yes (`agent { docker }`) | Yes (`container:` resource) | Yes (native `docker` executor) |
| Matrix builds | `strategy.matrix` | `parallel:matrix` | `matrix {}` (declarative) | `strategy.matrix` | `matrix` on workflow job |
| Manual approval | Environment protection rules | `when: manual` | `input` step | Environment checks/approvals | `type: approval` job |
| Native secrets vault | Secrets (repo/org/env) | CI/CD Variables (protected/masked) | Credentials Plugin | Variable Groups / Key Vault | Contexts |
| Reusable pipeline logic | Reusable workflows, composite actions | `include`/`extends`, Components | Shared Libraries | Templates | Orbs |
| Marketplace/ecosystem | GitHub Marketplace (huge) | GitLab templates/components | Massive plugin ecosystem | Azure Marketplace (tasks) | Orb Registry |
| Local testing | `act` (3rd party) | `gitlab-runner exec` | Run Jenkinsfile via local Jenkins | Limited (no first-party) | `circleci local execute` (Docker jobs only) |
| Native Kubernetes | Via self-hosted runner or actions | Native Kubernetes executor | Kubernetes plugin | Kubernetes agent pool | `machine`/remote docker; k8s via orb |
| Free tier (approx., check current pricing) | Generous for public repos | Generous for public repos | Free (self-hosted, infra cost only) | Free minutes for small teams | Free tier with credit limits |
| Best fit | GitHub-hosted repos, broad ecosystem | All-in-one DevOps platform (SCM+CI+CD+security) | Maximum customization/on-prem/legacy | Enterprises on Azure DevOps/.NET shops | Fast setup, strong caching/parallelism |

---

## 8. Common Patterns, Side-by-Side

### 8.1 Caching Dependencies

```yaml
# GitHub Actions
- uses: actions/cache@v4
  with:
    path: node_modules
    key: npm-${{ hashFiles('package-lock.json') }}
```
```yaml
# GitLab CI
cache:
  key:
    files: [package-lock.json]
  paths: [node_modules/]
```
```yaml
# Azure Pipelines
- task: Cache@2
  inputs:
    key: 'npm | "$(Agent.OS)" | package-lock.json'
    path: node_modules
```
```yaml
# CircleCI
- restore_cache:
    keys: [deps-{{ checksum "package-lock.json" }}]
- run: npm ci
- save_cache:
    key: deps-{{ checksum "package-lock.json" }}
    paths: [node_modules]
```
```groovy
// Jenkins (no first-class cache keyword; typically a plugin or manual stash/unstash + external cache dir)
stash name: 'node_modules', includes: 'node_modules/**'
```

### 8.2 Matrix / Parallel Testing

```yaml
# GitHub Actions
strategy:
  matrix:
    node: [18, 20, 22]
```
```yaml
# GitLab CI
parallel:
  matrix:
    - NODE_VERSION: ["18", "20", "22"]
```
```yaml
# Azure Pipelines
strategy:
  matrix:
    node18: { nodeVersion: '18.x' }
    node20: { nodeVersion: '20.x' }
```
```yaml
# CircleCI
matrix:
  parameters:
    node-version: ["18.20", "20.10", "22.1"]
```
```groovy
// Jenkins Declarative
matrix {
    axes {
        axis { name 'NODE_VERSION'; values '18', '20', '22' }
    }
    stages { stage('Test') { steps { sh "nvm use ${NODE_VERSION} && npm test" } } }
}
```

### 8.3 Manual Approval Gate

```yaml
# GitHub Actions — via environment protection rules (configured in repo Settings, not YAML)
jobs:
  deploy:
    environment: production   # reviewers required before this job runs
```
```yaml
# GitLab CI
deploy:
  when: manual
```
```groovy
// Jenkins
input message: 'Deploy to production?', ok: 'Deploy'
```
```yaml
# Azure Pipelines — via Environment "Checks" (configured in UI) attached to a deployment job
- deployment: DeployProd
  environment: production
```
```yaml
# CircleCI
- hold: { type: approval, requires: [test] }
```

### 8.4 Passing Artifacts Between Jobs

```yaml
# GitHub Actions
- uses: actions/upload-artifact@v4
  with: { name: dist, path: dist/ }
# ...in a later job:
- uses: actions/download-artifact@v4
  with: { name: dist }
```
```yaml
# GitLab CI (artifacts auto-flow to later stages)
build:
  artifacts:
    paths: [dist/]
```
```yaml
# Azure Pipelines
- task: PublishBuildArtifacts@1
  inputs: { PathtoPublish: dist, ArtifactName: drop }
# ...later:
- download: current
  artifact: drop
```
```yaml
# CircleCI
- persist_to_workspace: { root: ., paths: [dist] }
# ...in a later job:
- attach_workspace: { at: . }
```
```groovy
// Jenkins
stash name: 'dist', includes: 'dist/**'
// ...later stage:
unstash 'dist'
```

### 8.5 Docker Build & Push

```yaml
# GitHub Actions
- uses: docker/login-action@v3
  with: { username: ${{ secrets.DOCKER_USER }}, password: ${{ secrets.DOCKER_PASS }} }
- uses: docker/build-push-action@v6
  with: { push: true, tags: myorg/app:${{ github.sha }} }
```
```yaml
# GitLab CI
build-image:
  image: docker:24
  services: [docker:24-dind]
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```
```yaml
# Azure Pipelines
- task: Docker@2
  inputs:
    containerRegistry: 'my-acr-connection'
    repository: 'app'
    command: 'buildAndPush'
    tags: '$(Build.SourceVersion)'
```
```yaml
# CircleCI
- setup_remote_docker: { version: default }
- run: |
    docker build -t myorg/app:$CIRCLE_SHA1 .
    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
    docker push myorg/app:$CIRCLE_SHA1
```

### 8.6 Using Secrets

```yaml
# GitHub Actions
env:
  API_KEY: ${{ secrets.API_KEY }}
```
```yaml
# GitLab CI (masked/protected variable set in UI)
script:
  - curl -H "Authorization: Bearer $API_KEY" ...
```
```groovy
// Jenkins
environment { API_KEY = credentials('api-key-id') }
```
```yaml
# Azure Pipelines (secret variable, or Key Vault-linked)
script: echo $(apiKey)
```
```yaml
# CircleCI (Context or Project Environment Variable)
- run: curl -H "Authorization: Bearer $API_KEY" ...
```

### 8.7 Reusable Config

| Tool | Mechanism | Grain |
|---|---|---|
| GitHub Actions | Reusable workflow (`workflow_call`) | Whole job(s) |
| GitHub Actions | Composite action | Set of steps |
| GitLab CI | `extends:` | Job template inheritance |
| GitLab CI | `include:` | Whole file/template import |
| Jenkins | Shared Library (`@Library`) | Custom steps/functions in Groovy |
| Azure Pipelines | `template:` | Steps/jobs/stages fragment |
| Azure Pipelines | `extends:` (pipeline-level) | Whole pipeline template, enforceable org-wide |
| CircleCI | Orb | Jobs + commands + executors, versioned & published |
| CircleCI | `commands:` | Local reusable step sequence |

---

## 9. CLI Cheat Sheet

| Task | GitHub (`gh`) | GitLab (`glab`) | Jenkins | Azure (`az pipelines`) | CircleCI (`circleci`) |
|---|---|---|---|---|---|
| Trigger a run | `gh workflow run <file>` | `glab ci run` | `curl .../build` or CLI `build` | `az pipelines run --name <n>` | (push-triggered; or API `v2/project/.../pipeline`) |
| List runs | `gh run list` | `glab ci status` | Web UI / `list-jobs` | `az pipelines runs list` | Web UI / API |
| View logs | `gh run view --log` | `glab ci trace <job>` | Console output (Web/CLI) | `az pipelines runs show` | `circleci` has no log tail; use API/UI |
| Validate config | *(no offline linter; push to test, or use `act`)* | `glab ci lint` | Jenkinsfile Linter (`curl` endpoint) | *(use VS Code extension / pipeline preview)* | `circleci config validate` |
| Rerun failed | `gh run rerun --failed` | `glab ci retry` | Rebuild via UI | `az pipelines runs list` + re-queue | Rerun via UI/API |
| Manage secrets | `gh secret set` | `glab variable set` (GitLab 16+) | Credentials UI/CLI groovy scripting | `az pipelines variable-group` | `circleci env` isn't native; use UI/API |

---

## 10. Cron / Schedule Syntax

All five platforms use standard 5-field cron (`minute hour day-of-month month day-of-week`), evaluated in **UTC** unless stated otherwise:

```
 ┌───────────── minute (0–59)
 │ ┌───────────── hour (0–23)
 │ │ ┌───────────── day of month (1–31)
 │ │ │ ┌───────────── month (1–12)
 │ │ │ │ ┌───────────── day of week (0–6, Sun=0)
 │ │ │ │ │
 * * * * *
```

| Example | Meaning |
|---|---|
| `0 0 * * *` | Every day at midnight UTC |
| `*/15 * * * *` | Every 15 minutes |
| `0 9 * * 1-5` | 09:00 UTC, Mon–Fri |
| `0 0 1 * *` | Midnight on the 1st of each month |

Platform notes:
- **GitHub Actions**: `schedule.cron`, always UTC, can be delayed under high load.
- **GitLab CI**: configured via UI ("Pipeline schedules"), also supports a friendly cron widget.
- **Jenkins**: supports `H` (hash) instead of a fixed value, e.g. `H 2 * * *`, to spread load across jobs — strongly preferred over fixed minutes.
- **Azure Pipelines**: `schedules: - cron: "..."` with `always: true|false` (run even without changes) and explicit `branches`.
- **CircleCI**: `triggers.schedule.cron` inside a `workflows` entry, combined with `filters.branches`.

---

## 11. Security Best Practices Checklist

- [ ] **Pin third-party actions/orbs/plugins** to a full commit SHA or exact version, not a floating tag like `@main`/`@latest`.
- [ ] **Use OIDC / workload identity federation** for cloud auth (AWS/Azure/GCP) instead of long-lived static credentials.
- [ ] **Scope tokens minimally** — e.g., GitHub Actions `permissions:` block set to `read` by default, elevated only where needed.
- [ ] **Never echo secrets** to logs; rely on the platform's built-in masking, and avoid `set -x`/`echo $SECRET` patterns.
- [ ] **Protect deploy branches/environments** with required reviewers (GitHub Environments, GitLab protected branches + `when: manual`, Azure Environment checks).
- [ ] **Isolate untrusted PR triggers** — be careful with `pull_request_target` (GitHub) or pipelines triggered by forks, which can run with elevated secrets access.
- [ ] **Limit self-hosted runner exposure** — don't run self-hosted runners on public repos without additional isolation (ephemeral runners, network segmentation).
- [ ] **Validate/lint pipeline config** before merging (`glab ci lint`, `circleci config validate`, Jenkinsfile linter, `az pipelines` preview).
- [ ] **Use dependency/cache integrity checks** — lockfiles (`package-lock.json`, etc.) and checksums in cache keys to avoid poisoned caches.
- [ ] **Rotate credentials** stored in secret stores regularly, and audit who can view/modify them.
- [ ] **Fail fast, but fail loud** — surface clear errors and notifications (Slack/email/Teams) on pipeline failure, especially for deploy stages.
- [ ] **Keep pipelines DRY** via the reuse mechanism native to your platform (reusable workflows, `extends`/`include`, Shared Libraries, templates, orbs) to avoid config drift across many files.

---

## 12. Glossary

- **Pipeline** — the full definition of automated steps triggered by an event.
- **Workflow** (GH Actions/CircleCI) — an individually-triggerable pipeline definition; a repo can have several.
- **Job** — a set of steps executed together, usually on one machine/container.
- **Stage** — a logical phase (e.g., build/test/deploy); in Jenkins, "stage" = job-equivalent; in GitLab/Azure, stages group jobs.
- **Step/Task** — the smallest executable unit (a shell command or a pre-packaged action/task).
- **Runner/Agent/Executor** — the machine or container that actually executes jobs.
- **Artifact** — a file produced by a job, stored for later download or for use by another job/stage.
- **Cache** — a best-effort, reusable store (e.g., dependency folders) to speed up repeated runs; not guaranteed to persist.
- **Matrix build** — running the same job definition multiple times with different variable combinations (e.g., across OS/language versions).
- **Trigger** — the event that starts a pipeline: push, PR/MR, schedule, manual, API/webhook.
- **Environment** — a named deployment target (staging/production) that can carry protection rules, history, and approvals.
- **Secret/Variable** — sensitive or configurable values injected into jobs without hardcoding them in the config file.
- **DAG (Directed Acyclic Graph) pipeline** — jobs run as soon as their explicit dependencies finish, rather than waiting for an entire previous stage (GitLab `needs`, GH Actions `needs`, CircleCI `requires`).
- **OIDC (OpenID Connect)** — a protocol letting CI jobs exchange a short-lived token for cloud credentials without storing long-lived secrets.
- **Orb** (CircleCI) / **Action** (GitHub) / **Plugin** (Jenkins) / **Task** (Azure) / **Component/Template** (GitLab) — the platform-specific term for a shareable, reusable unit of CI functionality.

---

*This reference card covers the core mental model and most-used syntax for each platform. Always check each vendor's official docs for the newest task/action/orb versions, as these evolve frequently:*
- *GitHub Actions: docs.github.com/actions*
- *GitLab CI/CD: docs.gitlab.com/ee/ci*
- *Jenkins: jenkins.io/doc*
- *Azure Pipelines: learn.microsoft.com/azure/devops/pipelines*
- *CircleCI: circleci.com/docs*
