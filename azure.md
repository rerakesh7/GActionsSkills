# Azure — GitHub Actions Reference

## OIDC Authentication (preferred over client secrets)

### One-time setup
In Azure Portal or via CLI:
```bash
# Create app registration + federated credential
az ad app create --display-name "github-actions-deploy"
az ad sp create --id <app-id>

az ad app federated-credential create \
  --id <app-id> \
  --parameters '{
    "name": "github-main",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:YOUR_ORG/YOUR_REPO:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Assign role
az role assignment create \
  --assignee <app-id> \
  --role Contributor \
  --scope /subscriptions/<subscription-id>
```

### Workflow snippet
```yaml
permissions:
  id-token: write
  contents: read

steps:
  - name: Azure Login
    uses: azure/login@a65d910e8af852a8061c627c456678983e180302  # v2.2.0
    with:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

## ACR — Push Docker image

```yaml
  - name: Build and push to ACR
    uses: azure/docker-login@15c4aadf093404726ab2ff205b2cdd33fa6d054c  # v2
    with:
      login-server: ${{ vars.ACR_LOGIN_SERVER }}
      username: ${{ secrets.ACR_USERNAME }}
      password: ${{ secrets.ACR_PASSWORD }}

  - name: Build and push
    env:
      IMAGE: ${{ vars.ACR_LOGIN_SERVER }}/my-app:${{ github.sha }}
    run: |
      docker build -t $IMAGE .
      docker push $IMAGE
      echo "image=$IMAGE" >> $GITHUB_OUTPUT
```

---

## AKS — Deploy

```yaml
  - name: Set AKS context
    uses: azure/aks-set-context@37399a50c3544f602d9f2b2a7a9f15e545b8d791  # v4.0.0
    with:
      resource-group: my-rg
      cluster-name: my-cluster

  - name: Deploy to AKS
    run: |
      kubectl set image deployment/my-app my-app=${{ steps.build.outputs.image }}
      kubectl rollout status deployment/my-app
```

---

## App Service — Deploy

```yaml
  - name: Deploy to Azure App Service
    uses: azure/webapps-deploy@de617f46172a906d0617bb0e50d81e9e3c3ec720  # v3.0.1
    with:
      app-name: my-app
      images: ${{ steps.build.outputs.image }}
```

---

## Terraform state backend (Azure Storage)

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "mytfstate"
    container_name       = "tfstate"
    key                  = "production.terraform.tfstate"
  }
}
```

Secrets needed:
| Secret | Value |
|--------|-------|
| `AZURE_CLIENT_ID` | App registration client ID |
| `AZURE_TENANT_ID` | Azure AD tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Subscription ID |
| `TF_STORAGE_ACCOUNT` | Storage account for Terraform state |
