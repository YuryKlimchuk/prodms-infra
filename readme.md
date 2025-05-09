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
--set operator.watchNamespace="prodms"
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
      scramCredentialsSecretName: mongodb-creds-details
  additionalMongodConfig:
    storage.wiredTiger.engineConfig.journalCompressor: zlib
  statefulSet:
    spec:
      volumeClaimTemplates:
        - metadata:
            name: data-volume
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 1Gi
        - metadata:
            name: logs-volume
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 1Gi


# the user credentials will be generated from this secret
# once the credentials are generated, this secret is no longer required
---
apiVersion: v1
kind: Secret
metadata:
  name: prodms-mongo-db-creds
  namespace: prodms-infra
type: Opaque
stringData:
  password: mongodb-pwd
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mongodb-database
  namespace: prodms-infra
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: mongodb-database
  namespace: prodms-infra
rules:
  - apiGroups:
      - ""
    resources:
      - secrets
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - patch
      - delete
      - get
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: mongodb-database
  namespace: prodms-infra
subjects:
  - kind: ServiceAccount
    name: mongodb-database
roleRef:
  kind: Role
  name: mongodb-database
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-external-svc
  namespace: prodms-infra
  labels:
    app: prodms-mongodb-svc
spec:
  type: NodePort
  ports:
    - name: http
      port: 27017
      protocol: TCP
      nodePort: 32017
  selector:
    app: prodms-mongodb-svc

```

# kafka for warehouse

1. Install operator

```bash

helm install strimzi-kafka-cluster-operator oci://quay.io/strimzi-helm/strimzi-kafka-operator --namespace kafka-operator --create-namespace \
--set=watchAnyNamespace=true
```

2. Apply CRs

```yaml

---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: prodms-kafka-cluster-dual-role
  namespace: prodms-infra
  labels:
    strimzi.io/cluster: prodms-kafka-cluster
spec:
  replicas: 1
  roles:
    - controller
    - broker
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 1Gi
        deleteClaim: false
        kraftMetadata: shared
---

apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: prodms-kafka-cluster
  namespace: prodms-infra
  annotations:
    strimzi.io/node-pools: enabled
    strimzi.io/kraft: enabled
spec:
  kafka:
    version: 3.9.0
    metadataVersion: 3.9-IV0
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
      - name: external
        port: 9094
        type: nodeport
        tls: false
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      default.replication.factor: 1
      min.insync.replicas: 1
  entityOperator:
    topicOperator:
      resources:
        requests:
          memory: 256Mi
          cpu: 100m
        limits:
          memory: 512Mi
          cpu: 250m
    userOperator:
      resources:
        requests:
          memory: 256Mi
          cpu: 75m
        limits:
          memory: 512Mi
          cpu: 250m
```


# Minio for files

1. Add helm repository

```bash

helm repo add minio-operator https://operator.min.io
```

2. Install chart for operator

```bash

helm install \
  --namespace minio-operator \
  --create-namespace \
  minio-operator-release minio-operator/operator \
  --set operator.replicaCount=1
```

3. Verify installation

```bash

kubectl get pods -n minio-operator --watch
```

4. Install tenant chart

```bash

helm install \
--namespace prodms-infra \
--create-namespace \
prodms-minio-release \
minio-operator/tenant \
--set tenant.name=prodms-minio \
--set tenant.pools[0].servers=1 \
--set tenant.pools[0].name=minio-pool-0 \
--set tenant.pools[0].volumesPerServer=1 \
--set tenant.pools[0].size="1Gi" \
--set tenant.certificate.requestAutoCert=false
```

5. Add initial data

```bash

kubectl exec --namespace prodms-infra prodms-minio-minio-pool-0-0 -it -- bash

cat <<'EOF' >> /tmp/test__as.txt
Content of tmp/test__as.txt file...
EOF
cat <<'EOF' >> /tmp/test__si.txt
Content of tmp/test__si.txt file...
EOF
cat <<'EOF' >> /tmp/test__ov.txt
Content of tmp/test__ov.txt file...
EOF

mc config host add minio http://localhost:9000 minio minio123 > /dev/null
mc cp --tags="type=as" tmp/test__as.txt minio/drawings-bucket/test__as.txt > /dev/null
mc cp --tags="type=si" tmp/test__si.txt minio/drawings-bucket/test__si.txt > /dev/null
mc cp --tags="type=ov" tmp/test__ov.txt minio/drawings-bucket/test__ov.txt > /dev/null
exit 0
```


# postgres for tech

```bash
helm upgrade --install prodms-postgres-tech --namespace prodms --create-namespace cnpg/cluster \
--set cluster.storage.size="1Gi" \
--set cluster.initdb.database="tech" \
--set cluster.instances=1 \
--set cluster.initdb.owner="pg-user"
```