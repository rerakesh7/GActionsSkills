# GCP — GitHub Actions Reference

## Workload Identity Federation (preferred over service account keys)

### One-time setup (Terraform)
```hcl
resource "google_iam_workload_identity_pool" "github" {
  workload_identity_pool_id = "github-pool"
}

resource "google_iam_workload_identity_pool_provider" "github" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-provider"
  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.repository" = "assertion.repository"
  }
  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }
}

resource "google_service_account_iam_member" "github_actions" {
  service_account_id = google_service_account.deploy.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.repository/YOUR_ORG/YOUR_REPO"
}
```

### Workflow snippet
```yaml
permissions:
  id-token: write
  contents: read

steps:
  - name: Authenticate to Google Cloud
    uses: google-github-actions/auth@55bd3a7c6e2ae7cf1877fd1ccb9d54c0503c457c  # v2.1.2
    with:
      workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
```

---

## Artifact Registry — Push Docker image

```yaml
  - name: Set up Cloud SDK
    uses: google-github-actions/setup-gcloud@98ddc00a17442e89a24bbf282954a3b65ce6d200  # v2.1.0

  - name: Configure Docker for Artifact Registry
    run: gcloud auth configure-docker ${{ vars.GCP_REGION }}-docker.pkg.dev

  - name: Build and push image
    env:
      IMAGE: ${{ vars.GCP_REGION }}-docker.pkg.dev/${{ vars.GCP_PROJECT }}/my-repo/my-app:${{ github.sha }}
    run: |
      docker build -t $IMAGE .
      docker push $IMAGE
      echo "image=$IMAGE" >> $GITHUB_OUTPUT
```

---

## Cloud Run — Deploy

```yaml
  - name: Deploy to Cloud Run
    uses: google-github-actions/deploy-cloudrun@d9d3789b0b9e07e1bec5b3f8b640008e58cd81e8  # v2.2.0
    with:
      service: my-app
      region: ${{ vars.GCP_REGION }}
      image: ${{ steps.build.outputs.image }}
```

---

## GKE — Deploy

```yaml
  - name: Get GKE credentials
    uses: google-github-actions/get-gke-credentials@6051de21ad50fbb1767bc93c11357a49082ad116  # v2.2.1
    with:
      cluster_name: my-cluster
      location: ${{ vars.GCP_REGION }}

  - name: Deploy to GKE
    run: |
      kubectl set image deployment/my-app my-app=${{ steps.build.outputs.image }}
      kubectl rollout status deployment/my-app
```

---

## Terraform state backend (GCS)

```hcl
terraform {
  backend "gcs" {
    bucket = "my-tf-state"
    prefix = "env/production"
  }
}
```

Secrets needed:
| Secret | Value |
|--------|-------|
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | Full WIF provider resource name |
| `GCP_SERVICE_ACCOUNT` | Service account email |
| `TF_STATE_BUCKET` | GCS bucket name |
