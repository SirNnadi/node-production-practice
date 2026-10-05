# Node Production Practice

A deliberately small Node.js Task API for practicing a realistic DevOps delivery and production-support workflow. It uses Node's built-in HTTP server and test runner, so there are no third-party npm dependencies.

## API endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/` | Service information |
| GET | `/health` | Liveness check |
| GET | `/ready` | Readiness check |
| GET | `/api/tasks` | List in-memory tasks |
| POST | `/api/tasks` | Create a task (`{"title":"..."} `) |
| DELETE | `/api/tasks/:id` | Delete a task |

Tasks are held in memory and reset when the process restarts. This is intentional for the first practice stage; a later exercise is to add a persistent database.

## 1. Run locally

Requirements: Node.js 20+ and npm.

```bash
git clone <YOUR_REPOSITORY_URL>
cd node-production-practice
npm ci
npm test
npm run build
npm start
```

In another terminal:

```bash
curl http://localhost:3000/health
curl http://localhost:3000/api/tasks
curl -X POST http://localhost:3000/api/tasks \
  -H 'content-type: application/json' \
  -d '{"title":"Practice production release"}'
```

## 2. Docker

```bash
docker build -t node-production-practice:1.0.0 .
docker run --rm -p 3000:3000 node-production-practice:1.0.0
curl http://localhost:3000/health
```

The container runs as a non-root user and includes a Docker health check.

## 3. GitHub workflow

Push this repository to GitHub. The workflow in `.github/workflows/ci.yml` runs tests, build validation, and a Docker image build on pull requests and pushes to `main`. It does not deploy to production.

## 4. ECR with Terraform

Review `terraform/ecr` before use. Configure AWS credentials through your approved AWS identity method, then:

```bash
cd terraform/ecr
terraform init
terraform fmt -check
terraform validate
terraform plan
```

Inspect the plan and get approval before `terraform apply`. Applying creates billable/managed AWS resources. After creation, use the `repository_url` output to tag and push your image. Never commit credentials or Terraform state.

## 5. Helm / EKS

The Helm chart is under `helm/node-production-practice`. Update the image repository and tag in a values file to match your pushed image. First render and inspect:

```bash
helm lint ./helm/node-production-practice
helm template node-app ./helm/node-production-practice \
  -f ./helm/node-production-practice/values-dev.yaml
```

After confirming the current Kubernetes context and namespace are correct, deploy to a non-production cluster/namespace:

```bash
kubectl config current-context
kubectl get nodes
helm upgrade --install node-app ./helm/node-production-practice \
  --namespace devops --create-namespace \
  -f ./helm/node-production-practice/values-dev.yaml
kubectl rollout status deployment/node-app -n devops
kubectl get pods,svc -n devops
kubectl port-forward svc/node-app 3000:80 -n devops
```

Then test `http://localhost:3000/health`. Do not deploy to a real production cluster without the team's approvals, image policy, secrets integration, resource sizing, ingress/TLS, monitoring, and rollback plan.

## 6. Production support exercises

1. Break the Docker image tag and diagnose `ImagePullBackOff`.
2. Set a low memory limit and inspect pod events / `OOMKilled`.
3. Change the readiness path and observe rollout behavior.
4. Scale replicas and verify rollout status.
5. Deploy a new image tag, inspect Helm history, and practice rollback in a test namespace.
6. Add an ExternalSecret backed by AWS Secrets Manager using a properly configured workload identity. Never place secret values in Git or Helm values.
7. Add persistence (e.g. PostgreSQL), migrations, and integration tests.
8. Add metrics, dashboards, alerting, and a runbook.
9. Extend CI to publish immutable images to ECR using short-lived AWS credentials (OIDC where supported), then add a gated deployment stage.

## Important limitations

This is a training starter, not a ready-to-use production service. It has in-memory storage, no authentication, no database, no ingress/TLS, and no production secret integration. The Helm default image is a placeholder; replace it with your own ECR URI and image tag before deploying. Review all infrastructure and deployment changes before applying them.
# node-production-practice
