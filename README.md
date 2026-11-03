# MiniShop GitOps repository

This repository owns Kubernetes manifests and the Argo CD Application. It contains no application source. Its `manifests/` directory is what Argo CD syncs. The application repository builds and publishes the frontend and backend images to GHCR; this repository selects the image tags to deploy.

## 1. Prepare this repository

This directory is its own local Git repository. Create a separate GitHub repository, for example `minishop-gitops`, and push this repository there when ready. Keep `argocd/application.yaml` and `manifests/` at its root. Replace `GHCR_USERNAME` in `argocd/application.yaml` with the GitHub owner of that repository. If your repository has another name, edit `spec.source.repoURL` too. Argo CD must have access to the repository.

The Argo CD Application points at `manifests/`, has no automated sync policy, and explicitly excludes `secret.example.yaml`. Keep real credentials out of the repository.

## 2. Create the namespace

```sh
kubectl create namespace minishop
```

If it already exists, continue.

## 3. Create the database Secret locally

Run this in your shell. It generates a password in memory and creates a Kubernetes Secret; it writes no password to Git. Use the same password for PostgreSQL and the API connection string.

```sh
DB_PASSWORD="$(openssl rand -hex 24)"
kubectl -n minishop create secret generic minishop-db \
  --from-literal=POSTGRES_PASSWORD="$DB_PASSWORD" \
  --from-literal=DATABASE_URL="Host=postgres;Port=5432;Database=minishop;Username=minishop;Password=$DB_PASSWORD"
unset DB_PASSWORD
```

`manifests/secret.example.yaml` only shows the required keys. Argo CD excludes that example from synchronization. For a real deployment, protect access to Kubernetes Secrets and etcd backups.

## 4. Configure image references

In `manifests/backend.yaml` and `frontend.yaml`, replace `GHCR_USERNAME` in the image names with your GHCR owner. Both files start at tag `v1`. Push a new tag and update these two references for later releases; manual sync lets you inspect each change before applying it.

## 5. Deploy with Argo CD

After this GitOps repository contains your edited files, is pushed to GitHub, and the two images are available to K3s:

```sh
kubectl apply -f argocd/application.yaml
kubectl -n argocd get application minishop
```

Open your existing Argo CD UI, select **minishop**, review the resources, and click **Sync**. Argo CD reads the separate repository; editing the local copy alone does not update the cluster. No Argo CD installation or cluster creation is needed.

## 6. Access the frontend

```sh
kubectl -n minishop port-forward service/frontend 3000:8080
```

Open <http://localhost:3000>. The **API connected** badge confirms that the frontend received the product list through Nginx and the backend Service.

## 7. Verify persistence and caching

Place an order in the browser, then inspect the database:

```sh
kubectl -n minishop exec postgres-0 -- psql -U minishop -d minishop -c 'SELECT id, product_id, created_at FROM orders ORDER BY id DESC LIMIT 5;'
kubectl -n minishop rollout restart deployment/backend
```

The order should still appear after the backend restarts. It should also remain after the PostgreSQL pod restarts because its StatefulSet uses a persistent volume. To check the Redis cache, request products once, then inspect the key and its remaining time:

```sh
kubectl -n minishop port-forward service/backend 8080:8080
# In another terminal:
curl http://localhost:8080/api/products
kubectl -n minishop exec deployment/redis -- redis-cli TTL minishop:products
```

The TTL should be between 0 and 30 seconds. To check fallback in the application repository’s local Compose stack, run `docker compose stop redis`, then `curl http://localhost:8080/api/products`; the API should still return products. Restore Redis with `docker compose start redis`.

## 8. Troubleshooting

| Symptom | Check |
| --- | --- |
| `ImagePullBackOff` | Confirm image owner/tag and that the packages are public or K3s has image pull credentials. |
| Backend does not become ready | Check `kubectl -n minishop logs deployment/backend` and `kubectl -n minishop get pods`. Verify the `minishop-db` Secret exists and its `DATABASE_URL` matches the PostgreSQL password. |
| PostgreSQL pod stays Pending | Check `kubectl -n minishop get pvc` and `kubectl -n minishop describe pvc postgres-data-postgres-0`. K3s needs a working default StorageClass. |
| Frontend shows API unavailable | Check the backend readiness probe and `kubectl -n minishop logs deployment/frontend`. Nginx expects the backend Service named `backend` on port 8080. |
| Redis is down | Products still load from PostgreSQL. Check `kubectl -n minishop logs deployment/redis`; cache entries are intentionally ephemeral. |
| Argo CD shows placeholder Secret | Confirm `spec.source.directory.exclude: secret.example.yaml` is present in the Application and refresh it. Never sync the example Secret. |

The API creates its two tables and seeds products on startup, so no first-deployment migration command is required. The seed uses `ON CONFLICT DO NOTHING`, so restarts do not duplicate products.

## Future CI/CD integration

A later pipeline in the application repository can publish images tagged with a commit SHA, then open a pull request here changing the two Deployment image tags. After reviewing and merging that pull request, manually sync the Argo CD Application. This keeps build history and deployment history in separate Git repositories.
