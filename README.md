# argocd-notifications

## Step 1: Create GitHub Personal Access Token

1. Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Click "Generate new token (classic)"
3. Give it a name like "ArgoCD Notifications"
4. Select scopes: repo:status (to update commit statuses)
5. Copy the generated token and save it securely


## Step 2: Create the Notification Secret

Create a secret to store your GitHub token:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
type: Opaque
stringData:
  github-token: "ghp_your_github_token_here"
```

Apply it: 
```
kubectl apply -f github-secret.yaml
```

## Step 3: Configure ArgoCD Notifications ConfigMap

Create or update the argocd-notifications-cm ConfigMap:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  # GitHub webhook service configuration
  service.webhook.github: |
    url: https://api.github.com/repos/YOUR_GITHUB_USERNAME/YOUR_REPO_NAME/statuses/{{.app.status.sync.revision}}
    headers:
    - name: Authorization
      value: token $github-token
    - name: Content-Type
      value: application/json

  # Templates for different deployment states
  template.app-deployed: |
    webhook:
      github:
        method: POST
        body: |
          {
            "state": "success",
            "description": "Application {{.app.metadata.name}} deployed successfully",
            "context": "argocd/{{.app.metadata.name}}",
            "target_url": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}"
          }

  template.app-sync-failed: |
    webhook:
      github:
        method: POST
        body: |
          {
            "state": "failure",
            "description": "Application {{.app.metadata.name}} sync failed",
            "context": "argocd/{{.app.metadata.name}}",
            "target_url": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}"
          }

  template.app-sync-running: |
    webhook:
      github:
        method: POST
        body: |
          {
            "state": "pending",
            "description": "Application {{.app.metadata.name}} sync in progress",
            "context": "argocd/{{.app.metadata.name}}",
            "target_url": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}"
          }

  # Triggers define when to send notifications
  trigger.on-deployed: |
    - description: Application is synced and healthy
      send:
      - app-deployed
      when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'

  trigger.on-sync-failed: |
    - description: Application sync failed
      send:
      - app-sync-failed
      when: app.status.operationState.phase in ['Error', 'Failed']

  trigger.on-sync-running: |
    - description: Application sync started
      send:
      - app-sync-running
      when: app.status.operationState.phase in ['Running']

  # ArgoCD server URL for links
  context: |
    argocdUrl: "https://your-argocd-server.com"
```

**Replace these values:**

YOUR_GITHUB_USERNAME/YOUR_REPO_NAME with your actual GitHub repo
https://your-argocd-server.com with your ArgoCD server URL

Apply the ConfigMap:
```
kubectl apply -f argocd-notifications-cm.yaml
```

## Step 4: Enable Notifications Controller

Make sure the ArgoCD notifications controller is running:
```
# Check if notifications controller is running
kubectl get pods -n argocd | grep notification

# If not running, you may need to enable it in your ArgoCD installation
# For example, if using Helm:
helm upgrade argocd argo/argo-cd --set notifications.enabled=true
```

## Step 5: Subscribe Your Applications

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

## Step 6: Test the Setup

1. Trigger a deployment by pushing changes to your repository or manually syncing in ArgoCD UI
2. Check GitHub - go to your repository and look for:
   . Status checks on commits (green checkmarks, red X's, or yellow dots)
   . The status will show "argocd/your-app-name"
3. Verify notifications are working:
```
# Check ArgoCD notification controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-notifications-controller
```


