# argocd-notifications

## Step 1: Create GitHub APP

## Step 2: Generate and download GitHub App key

<img width="586" height="168" alt="image" src="https://github.com/user-attachments/assets/98eacbae-0310-4861-a943-0e3af68e5849" />

## Step 3: Create the Notification Secret

Create a secret to store your GitHub token:
```bash
kubectl create secret generic argocd-notifications-secret \
  -n argocd \
  --from-file=githubAppPrivateKey=./argocd-notifications.2025-07-20.private-key.pem \
  --from-literal=githubAppID=123456 \
  --from-literal=githubAppInstallationID=123456789
```

## Step 3: Configure ArgoCD Notifications ConfigMap

Apply the ConfigMap:
```
kubectl apply -f argocd-notifications-cm.yaml
```

## Step 4: Enable Notifications Controller

Make sure the ArgoCD notifications controller is running:
```bash
kubectl get pods -n argocd | grep notification
```

## Step 5: Create deployment for the reference commit: 

```yaml
name: Deploy to Argo CD

on:
  workflow_dispatch:
  push:
    branches: [main]
    paths-ignore:
      - '.github/workflows/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Create GitHub Deployment
      id: create_deployment
      env:
        GITHUB_TOKEN: ${{ secrets.TOKEN }}
      run: |
        curl -X POST https://api.github.com/repos/${{ github.repository }}/deployments \
          -H "Authorization: token $GITHUB_TOKEN" \
          -H "Accept: application/vnd.github+json" \
          -d '{
            "ref": "${{ github.sha }}",
            "environment": "production",
            "description": "Argo CD deployment triggered",
            "required_contexts": [],
            "auto_merge": false
          }'
```

## Step 6: Subscribe Your Applications

Add notification subscriptions to your ArgoCD applications:
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  annotations:
    # Subscribe to deployment notifications
    notifications.argoproj.io/subscribe.on-deployed.github: ""
    notifications.argoproj.io/subscribe.on-health-degraded.github: ""
    notifications.argoproj.io/subscribe.on-sync-failed.github: ""
    notifications.argoproj.io/subscribe.on-sync-running.github: ""
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPO_NAME
    targetRevision: HEAD
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
