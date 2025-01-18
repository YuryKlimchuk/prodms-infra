# Postgres for archive
1. Install CloudNativePG

```bash

kubectl apply --server-side -f \
https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.25/releases/cnpg-1.25.0.yaml
```

2. Verify installation

```bash

kubectl get deployment -n cnpg-system cnpg-controller-manager
```

3. Add cnpg repo to helm

```bash

helm repo add cnpg https://cloudnative-pg.github.io/charts
```

4. Install cluster by helm chart

```bash

helm upgrade --install prodms-postgres-archive --namespace prodms-infra --create-namespace cnpg/cluster \
--set cluster.storage.size="1Gi" \
--set cluster.instances=1 \
--set cluster.initdb.database="archive" \
--set cluster.initdb.owner="pg-user"
```

Run Helm Tests:
helm test --namespace prodms-infra prodms-postgres-archive

Get a list of all base backups:
kubectl --namespace prodms-infra get backups --selector cnpg.io/cluster=prodms-postgres-archive-cluster

Connect to the cluster's primary instance:
kubectl --namespace prodms-infra exec --stdin --tty services/prodms-postgres-archive-cluster-rw -- bash

