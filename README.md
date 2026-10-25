# cicd-app GitOps

Deployment configuration for the Flask CI/CD demo application. This repository stores the desired Kubernetes state in a Helm chart and is watched by Argo CD.

## Repository layout

```text
.
├── argocd/
│   └── application.yaml
└── charts/
    └── cicd-app/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── deployment.yaml
            └── service.yaml
```

## Deployment flow

1. The application repository builds, scans, and publishes an image to GHCR.
2. After approval in the `production` GitHub Environment, the application workflow updates `charts/cicd-app/values.yaml` to use the new image's Git commit SHA.
3. Argo CD watches this repository and automatically reconciles the Helm chart into the `cicd-app` namespace.
4. Kubernetes performs the rollout; readiness and liveness checks are configured in the Deployment.

Application repository: https://github.com/Abhishekk-20/cicd-app

## Inspect current state

```bash
grep -n '^  tag:' charts/cicd-app/values.yaml
helm lint charts/cicd-app
helm template cicd-app charts/cicd-app
kubectl get application cicd-app -n argocd
kubectl get deployment,pods,svc -n cicd-app
kubectl rollout status deployment/cicd-app -n cicd-app
```

Compare the image tag in `values.yaml` with the live Deployment:

```bash
kubectl get deployment cicd-app -n cicd-app \
  -o=jsonpath='{.spec.template.spec.containers[0].image}{"\\n"}'
```

For a healthy, converged deployment, the Argo CD Application should report `Synced` and `Healthy`.

## Manual rollback

A rollback is performed by reverting the Git commit that changed the image tag. First confirm the working tree is clean and inspect the recent deployment commits:

```bash
git status --short
git log -5 --oneline -- charts/cicd-app/values.yaml
```

Choose the commit that introduced the undesired image tag, then:

```bash
git revert <bad-deployment-commit>
git push origin main
```

Do not blindly revert the most recent commit if it is already a revert or contains unrelated changes. Argo CD should detect the restored tag and reconcile Kubernetes. Verify the live image and rollout with the commands above.

## Notes

- `values.yaml` is the source of truth for the desired image tag and chart settings.
- Keep credentials and tokens out of this repository.
- The application image must be pullable by the Kubernetes cluster. Use a public GHCR package or configure an image pull secret for a private package.
- The rollback shown here is a manual Git revert; it does not automatically select a previous healthy release.
