---
name: setup-github-actions
description: Scaffold a GitHub Actions CI/CD pipeline tailored to your stack, cloud provider, and environments. Use this skill whenever a user wants to set up CI/CD, automate deployments, create a pipeline, add GitHub Actions workflows, configure build/test/deploy automation, or asks "how do I deploy with GitHub Actions". Also trigger when the user mentions needing lint, test, build, or deploy steps automated — even if they don't say "GitHub Actions" explicitly.
---

# Setup GitHub Actions

Scaffold a production-ready GitHub Actions CI/CD pipeline through an interactive interview. The goal is a working `.github/workflows/` setup the user can commit immediately — not a generic template.

## Process Overview

1. **Interview** — Understand the stack, cloud targets, and environments
2. **Plan** — Propose a workflow structure for confirmation before writing any files
3. **Scaffold** — Generate all workflow YAML files plus supporting scripts/configs
4. **Explain** — Walk through secrets that need to be configured and next steps

---

## Step 1: Interview

Ask only what you don't already know from context. If the repo is available, explore it first (`ls`, `cat package.json`, `cat Dockerfile`, etc.) to infer as much as possible before asking.

Gather:

- **Language/runtime**: Node, Python, Go, Java, etc.
- **Build tool**: npm/yarn/pnpm, Maven, Gradle, pip, etc.
- **Test framework**: Jest, pytest, JUnit, etc. — or "no tests yet"
- **Containerized?**: Dockerfile present? Image registry target (ECR, GCR, ACR, GHCR)?
- **Cloud provider(s)**: AWS, GCP, Azure — or multi-cloud
- **IaC tool**: Terraform, Pulumi, CDK, or none
- **Environments**: e.g. dev → staging → production, or just main → prod
- **Deployment target**: ECS, GKE, AKS, Lambda, App Service, Cloud Run, static S3/GCS, etc.
- **Branch strategy**: trunk-based, GitFlow, or other
- **Existing secrets/OIDC**: Are cloud credentials already set up, or does this need OIDC configuration too?

Don't ask all at once — keep it conversational. Two or three questions at a time is fine.

---

## Step 2: Plan

Before writing any files, present a concise plan:

```
Proposed workflows:
1. ci.yml       — runs on every PR: lint → test → build
2. deploy.yml   — runs on merge to main: build image → push to registry → deploy to staging
3. promote.yml  — manual trigger: promote staging image to production
```

Ask: "Does this look right, or do you want to adjust anything?"

Only proceed once confirmed.

---

## Step 3: Scaffold

Generate files into `.github/workflows/`. Follow these principles:

### General
- Pin all third-party actions to a full commit SHA (not `@v3`) for supply chain safety
- Use `GITHUB_TOKEN` for GHCR; use OIDC (not long-lived keys) for AWS/GCP/Azure where possible
- Add `concurrency` groups to cancel redundant runs on the same branch
- Use `workflow_dispatch` inputs for any manual trigger workflows
- Cache dependencies (npm, pip, gradle, etc.) to speed up runs
- Keep jobs focused — one job per concern; use `needs:` for sequencing

### Multi-cloud patterns
Load the relevant reference file for each cloud target:
- AWS → read `references/aws.md`
- GCP → read `references/gcp.md`
- Azure → read `references/azure.md`

If multi-cloud, read all relevant files.

### Terraform integration
If IaC tool is Terraform, include a `terraform.yml` workflow:
- `terraform fmt -check` and `terraform validate` on PRs
- `terraform plan` on PRs with output posted as a PR comment
- `terraform apply` only on merge to main, with environment protection rules
- Use remote state (S3 + DynamoDB, GCS, or Azure Storage) — ask which if unknown
- Never run `apply` without a prior `plan` artifact

### Structure example

```
.github/
  workflows/
    ci.yml
    deploy.yml
    promote.yml        # if multi-env
    terraform.yml      # if IaC = Terraform
```

---

## Step 4: Explain

After scaffolding, always produce a **"What to do next"** section:

```markdown
## What to do next

### Secrets to configure in GitHub (Settings → Secrets → Actions)
| Secret name         | Description                        |
|---------------------|------------------------------------|
| AWS_ROLE_ARN        | IAM role ARN for OIDC auth         |
| TF_STATE_BUCKET     | S3 bucket name for Terraform state |

### Branch protections to enable
- Require `ci` workflow to pass before merge on `main`
- Require `terraform plan` check on PRs touching `infra/`

### First run
1. Push a branch and open a PR — CI should trigger
2. Merge to main — deploy to staging should trigger
3. Review staging, then run the `promote` workflow manually
```

---

## Quality Checklist

Before finishing, verify the generated workflows:
- [ ] No hardcoded credentials or account IDs
- [ ] All third-party actions pinned to SHA
- [ ] `concurrency` group set to avoid duplicate runs
- [ ] Secrets documented in "What to do next"
- [ ] Terraform `apply` protected by environment approval (if applicable)
- [ ] Cache steps present for dependencies
- [ ] Each job has a clear `name:` for readability in the Actions UI
