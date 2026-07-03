# ArgoCD Migration - JIRA Tickets

## TICKET 1: Ops Infra Setup for ArgoCD Migration

### Title
Ops Infra Setup for ArgoCD Migration

### Description
Create or update all required ops-side resources for the application migration: AppSets in the eks repo, AppProject access and permissions, ExternalSecrets IAM role and SecretStore configuration, and verification that the GitOps and ArgoCD GitHub Apps are installed.

### Purpose
Provides the infrastructure and access prerequisites needed for ArgoCD to discover, access, and deploy the application safely.

### Acceptance Criteria
- [ ] Create or update required AppSets in eks repository for target cluster environments
  - [ ] Default/standard AppSet configured
  - [ ] Preview environments AppSet configured for non-prod where applicable
- [ ] Confirm app namespace and cluster mappings are correct for dev, stg, and prod
- [ ] Create or update ArgoCD AppProject in eks repo
  - [ ] Project destinations include required namespace(s)
  - [ ] Source repositories allow application repo access
  - [ ] Team AD group mapped to required ArgoCD role permissions (sync/view/logs/exec as needed)
- [ ] Verify OneLogin/AD access path for team members to ArgoCD tile
- [ ] Configure ExternalSecrets IAM role for application namespace
  - [ ] AWS Secrets Manager path prefixes set correctly for envs
  - [ ] ESO assumer role relationship configured
- [ ] Configure SecretStore resource for namespace and verify it binds to the IAM role
- [ ] Verify GitHub Apps are installed and authorized at org/repo level
  - [ ] Allow GitOps GH App
  - [ ] ArgoCD GH App
- [ ] Validate resources are visible and healthy
  - [ ] AppSets visible in ArgoCD namespace
  - [ ] AppProject present and policy rules effective
  - [ ] ExternalSecret test sync succeeds for at least one secret path

### Dependency
Mostly external delay from DevOps/SRE, INFSYS, or GitHub org administration

### Blockers / Dependencies
- DevOps/SRE implementation and review timelines
- INFSYS group/access provisioning timeline
- GitHub org admin permissions and approval timelines

---

## TICKET 2: Set up Helm Charts and Verify Chart Compatibility

### Title
Set up Helm Charts and Verify Chart Compatibility

### Description
As a developer, I need to set up the Helm chart structure for cp-panes deployment to prepare for ArgoCD deployment.

### Acceptance Criteria
- [ ] Create `helm/` directory at repository root
- [ ] Create `helm/Chart.yaml` defining chart metadata and dependencies
- [ ] Create `helm/shared-values.yaml` with common configuration values
- [ ] Create environment-specific values files:
  - [ ] `helm/dev-values.yaml` for development environment
  - [ ] `helm/stg-values.yaml` for staging environment
  - [ ] `helm/prod-values.yaml` for production environment
- [ ] Update `Chart.yaml` to reference compatible chart versions:
  - [ ] web-app >= 3.0.0 (or verify current version compatibility)
  - [ ] Any other 2U in-house charts used by the application
- [ ] Validate Helm chart syntax with `helm lint` command
- [ ] All environment-specific values are properly configured
- [ ] Confirm ECR registry and image tag references are correct
- [ ] Create `helm/templates/` directory for custom Kubernetes configurations if needed

### Technical Notes
- The Helm chart will be referenced by ArgoCD AppSets in the ops repository
- ECR image tag will be updated automatically by Release Please workflow
- Each values file must include database URLs, secrets references, and environment-specific configurations
- Use AWS Secrets Manager references via ExternalSecrets pattern where applicable


---

## TICKET 3: Configure Release Please and Conventional Commits

### Title
Configure Release Please and Conventional Commits

### Description
As a developer, I need to configure Release Please for automated versioning and changelog generation based on conventional commits. This enables automated release management in the GitOps workflow.

### Acceptance Criteria
- [ ] Create `release-please-config.json` at repository root
  - [ ] Set initial-version to appropriate value (e.g., "0.1.0")
  - [ ] Configure release-type as "node"
  - [ ] Configure changelog-path as "CHANGELOG.md"
  - [ ] Set include-v-in-tag to false
- [ ] Create `.release-please-manifest.json` to track component versions
- [ ] Configure extra-files in release-please-config.json to update:
  - [ ] `helm/dev-values.yaml` with ECR image tag
  - [ ] `helm/stg-values.yaml` with ECR image tag
  - [ ] `helm/prod-values.yaml` with ECR image tag
  - [ ] `package.json` version field
- [ ] Verify package.json name matches application name in gitops.yaml
- [ ] Document conventional commit message format in CONTRIBUTING.md or README
- [ ] Team members trained on conventional commit messages:
  - [ ] `feat:` for new features (minor version bump)
  - [ ] `fix:` for bug fixes (patch version bump)
  - [ ] `feat!:`, `fix!:` for breaking changes (major version bump)
- [ ] Create CHANGELOG.md if not already present
- [ ] Validate release-please configuration JSON syntax

### Technical Notes
- Release Please will automatically create Release PRs when conventional commits are detected
- Version bumps are determined by commit message prefixes
- The configuration uses jsonpath to update ECR image tags in Helm values files
- Release Please integrates with GitHub to create releases and update tags

---

## TICKET 4: Create GitOps Configuration and GitHub Workflows

### Title
Create GitOps Configuration and GitHub Workflows

### Description
As a developer, I need to create the GitOps configuration file and GitHub Actions workflows to enable ArgoCD deployment and automated CI/CD for the cp-panes application.

### Acceptance Criteria
- [ ] Create `gitops.yaml` at repository root with:
  - [ ] Application name matching package.json name (verify Buildkite release_name if migrating)
  - [ ] Three app configurations for dev, stg, and prod environments
  - [ ] Correct cluster group (main) and namespace configuration
  - [ ] Helm chart paths and values file references
  - [ ] Preview environment configuration for dev environment
  - [ ] Correct image key and host key for preview deployments
  - [ ] Proper revision format for version tracking
- [ ] Create `.github/workflows/` directory
- [ ] Add `pull-request-single-app.yml` workflow with:
  - [ ] Trigger on PR open, synchronize, and reopen
  - [ ] Path filters for Dockerfile, helm/**, src/**, package.json
  - [ ] Configuration for docker build, push, and Helm validation
  - [ ] ECR repository naming convention (ops/cp-panes or similar)
  - [ ] Preview URL template for dev environment
  - [ ] Reference to gha-workflows repository workflow
- [ ] Add `release.yml` workflow:
  - [ ] Trigger on push to main branch
  - [ ] Integration with Release Please for PR creation and management
  - [ ] Automated changelog generation
  - [ ] GitHub release creation
- [ ] Add `promote-release.yml` workflow:
  - [ ] Manual workflow dispatch trigger
  - [ ] Promotion from one environment to another
  - [ ] gitops.yaml update with new version
  - [ ] ArgoCD sync trigger
- [ ] Verify GitHub Apps are installed:
  - [ ] Allow GitOps GH App (for deployment file updates)
  - [ ] ArgoCD GH App (for reading revisions and deploying)
- [ ] Validate all workflow YAML syntax
- [ ] Set required GitHub Actions secrets:
  - [ ] GITOPS_BOT_APP_ID (if not already set)
  - [ ] GITOPS_BOT_PRIVATE_KEY (if not already set)
  - [ ] AWS credentials (coordinate with SRE/DevOps team)

### Technical Notes
- GitHub workflows reference 2uinc/gha-workflows repository for reusable workflow logic
- The PR workflow creates preview environments for testing before merge
- Release workflow automates semantic versioning and changelog updates
- Promote workflow handles manual promotion between dev → stg → prod
- All workflows use GitHub Actions secrets managed by the organization

### gitops.yaml Structure
```yaml
apps:
  - name: cp-panes
    clusterGroup: main
    clusterEnv: stg
    env: dev
    namespace: ops
    revision: 0.1.0
    repoPath: helm
    sharedValues: shared-values.yaml
    values: dev-values.yaml
    previewEnv:
      enabled: 'true'
      baseIngress: dev.devops.2u.com
      imageKey: web-app.ecrTag
      hostKey: web-app.ingress.host
  # Similar configs for stg and prod...
```

---

## TICKET 5: Testing, Validation, and Go-Live Preparation

### Title
Testing, Validation, and Go-Live Preparation for ArgoCD

### Description
As a developer, I need to thoroughly test and validate the ArgoCD migration setup before enabling production deployments. This includes testing all workflows, verifying deployments, and documenting the new workflow.

### Acceptance Criteria
- [ ] **Workflow Testing:**
  - [ ] Create a test PR with conventional commit messages
  - [ ] Verify pull-request workflow executes successfully
  - [ ] Confirm Docker image builds and pushes to ECR
  - [ ] Verify Helm chart validates without errors
  - [ ] Confirm preview environment deploys correctly
  - [ ] Verify preview environment is accessible and functional
- [ ] **Release Testing:**
  - [ ] Merge test PR to main branch
  - [ ] Verify Release Please creates Release PR automatically
  - [ ] Confirm CHANGELOG.md is generated correctly
  - [ ] Verify version bump is calculated correctly (patch/minor/major)
  - [ ] Verify Helm values files are updated with new version
  - [ ] Merge Release PR and verify GitHub release is created
  - [ ] Verify git tag is created with correct naming convention
- [ ] **Deployment Testing:**
  - [ ] Verify ArgoCD detects gitops.yaml changes
  - [ ] Confirm application deploys to dev environment
  - [ ] Verify dev deployment is healthy and accessible
  - [ ] Test promotion workflow from dev to stg
  - [ ] Verify stg deployment succeeds and is healthy
  - [ ] Test promotion workflow from stg to prod (if applicable)
  - [ ] Verify prod deployment succeeds and is healthy
- [ ] **Documentation:**
  - [ ] Update README.md with new deployment workflow instructions
  - [ ] Document conventional commit message requirements
  - [ ] Document preview environment access instructions
  - [ ] Create runbook for common issues and troubleshooting
  - [ ] Document rollback procedures
  - [ ] Update team wiki/confluence with migration details
- [ ] **Validation:**
  - [ ] All environment variables and secrets properly configured
  - [ ] Application functionality verified in all environments
  - [ ] No breaking changes from Buildkite migration
  - [ ] Performance metrics are acceptable
  - [ ] Monitoring and logging are working correctly
  - [ ] All team members trained on new workflow
- [ ] **Sign-off:**
  - [ ] Development team approval
  - [ ] SRE/DevOps team confirmation
  - [ ] Product team verification

### Technical Notes
- Maintain Buildkite pipelines during testing period as fallback
- Test preview environments thoroughly before full deployment
- Verify all secrets are correctly referenced through ExternalSecrets
- Ensure database migrations (if any) are handled correctly
- Check monitoring alerts are triggered appropriately

### Blockers / Dependencies
- Tickets 1, 2, and 3 must be completed first
- SRE team confirmation that ArgoCD infrastructure is ready
- DevOps team must have configured all required AWS resources

---
