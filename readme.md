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



# Mongodb for warehouse

1. Add repo

```bash

helm repo add mongodb https://mongodb.github.io/helm-charts
```

2. Install operator by helm

```bash

helm install community-operator mongodb/community-operator --namespace mongodb-operator --create-namespace \
--set operator.watchNamespace="prodms-infra"
```

3. Load yaml

```yaml

---
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  namespace: prodms-infra
  name: prodms-mongodb
spec:
  members: 1
  type: ReplicaSet
  version: "6.0.5"
  security:
    authentication:
      modes: ["SCRAM"]
  users:
    - name: mongodb-user
      db: warehouse
      passwordSecretRef: # a reference to the secret that will be used to generate the user's password
        name: prodms-mongo-db-creds
      roles:
        - name: clusterAdmin
          db: warehouse
        - name: userAdminAnyDatabase
          db: warehouse
      scramCredentialsSecretName: my-scram
  additionalMongodConfig:
    storage.wiredTiger.engineConfig.journalCompressor: zlib

# the user credentials will be generated from this secret
# once the credentials are generated, this secret is no longer required
---
apiVersion: v1
kind: Secret
metadata:
  name: prodms-mongo-db-creds
type: Opaque
stringData:
  password: mongodb-pwd

```