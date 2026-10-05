# Cost Management self-managed — connected install

This guide installs the Red Hat Lightspeed cost management self-managed beta on a connected OpenShift 4.22 cluster. It records the procedure that produced a working install, written so you can repeat it on your own cluster. Values from the lab that proved the procedure are called out as **Lab example**. Substitute your API, domain, storage class, and identity provider.

The product article is [Knowledgebase 7148128](https://access.redhat.com/articles/7148128). The operator source repository is [project-koku/koku-server-operator](https://github.com/project-koku/koku-server-operator). That repository is not the install path. Install from the catalog image and from `redhat-operators`, as this guide does.

A cluster that cannot pull from the internet should follow [cost-management-airgapped.md](./cost-management-airgapped.md) first, then return here at the namespace step.

Do not commit passwords, client secrets, or private keys. Keep them in a file outside git.

## What you get

OpenShift usage cost for the cluster you point at this install. This beta does not ingest AWS, Azure, or GCP bills. Resource optimization stays off.

You install two operators:

| Operator | Where it comes from | Where it is installed |
|---|---|---|
| Cost Management Service Operator `0.0.1`, package `koku-service-operator`, channel `beta` | Catalog image `quay.io/project-koku/koku-service-operator-catalog:v0.0.1`. It is not in the default `redhat-operators` catalog. | `openshift-operators` only. The CSV allows AllNamespaces and no other install mode. At runtime the controller watches only its own namespace. |
| Cost Management Metrics Operator, package `costmanagement-metrics-operator`, channel `stable` (verified at `4.4.2`) | `redhat-operators` | Its own namespace, `cost-onprem`. AllNamespaces is not supported. |

The service does not deploy its database, cache, broker, object store, or identity provider when those fields are set to `deploy: false`. You provide:

- PostgreSQL 16
- a Redis-compatible cache
- Kafka
- an S3 API
- an OpenID Connect provider (this guide uses Keycloak)

## Values you fill in

Create a file of non-secret settings and source it in every new terminal. Change the right-hand sides. Leave the custom resource name as `cost-onprem`. Database names are that name with the hyphens removed (`costonprem_koku`, and so on). Renaming the custom resource means renaming those databases to match.

```bash
mkdir -p ~/cost-mgmt-rebuild
chmod 700 ~/cost-mgmt-rebuild
umask 077
cat > ~/cost-mgmt-rebuild/site.env <<'EOF'
# API listens on 6443. Port 443 on the same name is often the console.
API_URL=https://api.example.com:6443
APPS_DOMAIN=apps.example.com
# Name shown for this cluster in the cost UI. One cost model per source.
SOURCE_NAME=example.com
STORAGE_CLASS=your-storage-class
KC_HOST=keycloak.example.com
KC_REALM=kubernetes
ORG_ID=1000001
ACCOUNT_NUMBER=1000001
# URL that serves the PEM certificate of the CA that signed Keycloak.
CA_URL=https://idm.example.com/ipa/config/ca.crt
# Resolver that can answer the Keycloak hostname. Forward only that name.
DNS_UPSTREAM=192.0.2.10
NODE_NAME=replace-with-oc-get-nodes
EOF
set -a
source ~/cost-mgmt-rebuild/site.env
set +a
```

The UI will be published at:

`https://cost-onprem-ui-openshift-operators.${APPS_DOMAIN}`

The API gateway will be published at:

`https://cost-onprem-gateway-openshift-operators.${APPS_DOMAIN}`

**Lab example.** `API_URL=https://api.sno.dragon.internal:6443`, `APPS_DOMAIN=apps.sno.dragon.internal`, `SOURCE_NAME=sno.dragon.internal`, `STORAGE_CLASS=lvms-vg1`, `KC_HOST=kc01.dragon.internal`, `KC_REALM=kubernetes`, `ORG_ID=1000001`, `ACCOUNT_NUMBER=1000001`, `CA_URL=https://idm01.dragon.internal/ipa/config/ca.crt`, `DNS_UPSTREAM=192.168.1.10`, `NODE_NAME=00-0a-f7-a6-59-d8`. The node address used for the laptop hosts file was `192.168.1.238`. The cluster id was `9c882036-e1c7-4452-8a75-5b07ebb22b2a`.

## Order

1. Log in and check the CPU.
2. Create namespaces and secrets.
3. PostgreSQL, cache, Kafka, and S3.
4. Keycloak clients, user, and claims.
5. Cluster DNS for the identity provider.
6. Service operator.
7. UI image, including a rebuild when the CPU lacks x86-64-v3.
8. `CostManagementServiceConfig`.
9. Metrics operator.
10. Tenant, then the metrics service-account grant.
11. Metrics custom resource.
12. Log in and grant `costadmin` Default admin access.
13. Confirm an upload is kept, then add a price list and a cost model.

---

## 1. Log in

`oc` has to talk to the API, which listens on port `6443`. Port 443 on the API hostname is often the console. That port presents a certificate for `*.apps...` and returns HTML, and `oc login` then fails with "Seems you passed an HTML page".

```bash
set -a
source ~/cost-mgmt-rebuild/site.env
set +a
oc login -u kubeadmin "$API_URL"
```

Accept the insecure-certificate prompt when the API certificate is signed by a private CA. `oc whoami` should print `kube:admin`. Tokens last about a day. Do not store the kubeadmin password in git.

**Lab example.** `oc login -u kubeadmin https://api.sno.dragon.internal:6443`. `https://api.sno.dragon.internal` without `:6443` hit the console certificate for `*.apps.sno.dragon.internal`.

## 2. Check the CPU

RHEL 10 and UBI 10 images require x86-64-v3. On a CPU without AVX2 they exit immediately with `Fatal glibc error: CPU does not support x86-64-v3`. This check decides whether you use the published images or the c9s images and the UI rebuild later.

```bash
oc get nodes
oc debug "node/${NODE_NAME}" -- chroot /host lscpu | grep -E 'Model name|Flags'
```

If the Flags line contains `avx2`, you can use the image tags from the operator sample and skip the UI rebuild in section 7. If `avx2` is absent, use the c9s PostgreSQL and Redis images in this guide and build the `lab-c9s` UI image.

**Lab example.** Intel Xeon E5-2630 v2. Flags included `avx` and did not include `avx2`. The install used `postgresql-16-c9s`, `redis-7-c9s`, and a locally built UI tag `lab-c9s`.

## 3. Namespaces and secrets

`cost-deps` holds the database, cache, Kafka, and object storage that you run yourself. `cost-onprem` holds only the metrics operator. The service operator cannot be subscribed in its own namespace, and it only reconciles objects in the namespace it runs in, so its subscription, custom resource, and the secrets it reads all go in `openshift-operators`.

Generate passwords in this shell and save them. Later commands read the file. Do not put a password on the command line. That lands in shell history.

```bash
set -a
source ~/cost-mgmt-rebuild/site.env
set +a

export KOKU_PW="$(openssl rand -hex 24)"
export RBAC_PW="$(openssl rand -hex 24)"
export ROS_PW="$(openssl rand -hex 24)"
export KRUIZE_PW="$(openssl rand -hex 24)"
export POSTGRES_ADMIN_PW="$(openssl rand -hex 24)"
export REDIS_PW="$(openssl rand -hex 24)"
export S3_ACCESS_KEY="$(openssl rand -hex 16)"
export S3_SECRET_KEY="$(openssl rand -hex 32)"
export METRICS_CLIENT_UUID="$(uuidgen)"

cat > ~/cost-mgmt-rebuild/secrets.env <<EOF
KOKU_PW=$KOKU_PW
RBAC_PW=$RBAC_PW
ROS_PW=$ROS_PW
KRUIZE_PW=$KRUIZE_PW
POSTGRES_ADMIN_PW=$POSTGRES_ADMIN_PW
REDIS_PW=$REDIS_PW
S3_ACCESS_KEY=$S3_ACCESS_KEY
S3_SECRET_KEY=$S3_SECRET_KEY
METRICS_CLIENT_UUID=$METRICS_CLIENT_UUID
EOF
chmod 600 ~/cost-mgmt-rebuild/secrets.env

oc new-project cost-deps
oc new-project cost-onprem

oc create secret generic postgresql-auth -n cost-deps \
  --from-literal=postgres-password="$POSTGRES_ADMIN_PW" \
  --from-literal=koku-password="$KOKU_PW" \
  --from-literal=rbac-password="$RBAC_PW" \
  --from-literal=ros-password="$ROS_PW" \
  --from-literal=kruize-password="$KRUIZE_PW"

oc create secret generic valkey-auth -n cost-deps \
  --from-literal=password="$REDIS_PW"

oc create secret generic minio-auth -n cost-deps \
  --from-literal=access-key="$S3_ACCESS_KEY" \
  --from-literal=secret-key="$S3_SECRET_KEY"

oc create secret generic cost-db-credentials -n openshift-operators \
  --from-literal=koku-user=koku \
  --from-literal=koku-password="$KOKU_PW" \
  --from-literal=rbac-user=rbac \
  --from-literal=rbac-password="$RBAC_PW" \
  --from-literal=ros-user=ros \
  --from-literal=ros-password="$ROS_PW" \
  --from-literal=kruize-user=kruize \
  --from-literal=kruize-password="$KRUIZE_PW"

oc create secret generic cost-cache-credentials -n openshift-operators \
  --from-literal=redis-password="$REDIS_PW"

oc create secret generic cost-object-storage -n openshift-operators \
  --from-literal=access-key="$S3_ACCESS_KEY" \
  --from-literal=secret-key="$S3_SECRET_KEY"
```

The operator-facing secret uses different key names from the database secret: `koku-user` / `koku-password`, and the same pattern for `rbac`, `ros`, and `kruize`. The cache key the operator reads is `redis-password`. The object-storage keys are `access-key` and `secret-key`. User names inside `cost-db-credentials` are `koku`, `rbac`, `ros`, and `kruize`.

Resource optimization is disabled later, but the credentials secret still has `ros` and `kruize` entries because the operator expects those keys.

The identity-provider CA is created in section 4, after you have the PEM. The UI and metrics client secrets are created in that section too, after Keycloak issues them.

## 4. PostgreSQL 16

The service opens four databases. The names are the custom resource name with hyphens removed, plus the component: `costonprem_koku`, `costonprem_rbac`, `costonprem_ros`, and `costonprem_kruize`. Do not use a database named `postgres`.

If you already run PostgreSQL 16, create those roles and databases there and point the custom resource at that host. The StatefulSet below is the stand-in used when you do not.

On a CPU without AVX2, `quay.io/sclorg/postgresql-16-c10s` crashes. Use `quay.io/sclorg/postgresql-16-c9s:c9s`. Replace `STORAGE_CLASS` if you did not export `site.env` in this shell. The manifest below uses the variable.

```bash
set -a; source ~/cost-mgmt-rebuild/site.env; set +a

cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: postgresql
  namespace: cost-deps
spec:
  selector:
    app: postgresql
  ports:
    - name: postgresql
      port: 5432
      targetPort: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: cost-deps
spec:
  serviceName: postgresql
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
        - name: postgresql
          image: quay.io/sclorg/postgresql-16-c9s:c9s
          imagePullPolicy: IfNotPresent
          ports:
            - name: postgresql
              containerPort: 5432
          env:
            - name: POSTGRESQL_DATABASE
              value: costonprem_koku
            - name: POSTGRESQL_USER
              value: koku
            - name: POSTGRESQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgresql-auth
                  key: koku-password
            - name: POSTGRESQL_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgresql-auth
                  key: postgres-password
          readinessProbe:
            exec:
              command: ["pg_isready", "-h", "127.0.0.1", "-U", "koku", "-d", "costonprem_koku"]
            initialDelaySeconds: 10
            periodSeconds: 5
          resources:
            requests:
              cpu: 100m
              memory: 512Mi
            limits:
              memory: 2Gi
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
            runAsNonRoot: true
            seccompProfile:
              type: RuntimeDefault
          volumeMounts:
            - name: data
              mountPath: /var/lib/pgsql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: ${STORAGE_CLASS}
        resources:
          requests:
            storage: 30Gi
EOF

oc rollout status statefulset/postgresql -n cost-deps
```

A successful StatefulSet prints `partitioned roll out complete`. The image creates only `costonprem_koku`. Koku migration fails with `permission denied for pg_stat_statements` until `koku` is a superuser and the extension exists. Use `oc exec -i`. A TTY (`-it`) and a heredoc fight each other.

```bash
set -a; source ~/cost-mgmt-rebuild/secrets.env; set +a

oc exec -i -n cost-deps postgresql-0 -- \
  bash -lc 'export PGPASSWORD="$POSTGRESQL_ADMIN_PASSWORD"; psql -U postgres -v ON_ERROR_STOP=1 -d postgres' <<EOF
CREATE USER rbac WITH PASSWORD '${RBAC_PW}';
CREATE USER ros WITH PASSWORD '${ROS_PW}';
CREATE USER kruize WITH PASSWORD '${KRUIZE_PW}';
CREATE DATABASE costonprem_rbac OWNER rbac;
CREATE DATABASE costonprem_ros OWNER ros;
CREATE DATABASE costonprem_kruize OWNER kruize;
ALTER USER koku WITH SUPERUSER;
\c costonprem_koku
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
ALTER SCHEMA public OWNER TO koku;
\c costonprem_rbac
ALTER SCHEMA public OWNER TO rbac;
\c costonprem_ros
ALTER SCHEMA public OWNER TO ros;
\c costonprem_kruize
ALTER SCHEMA public OWNER TO kruize;
EOF
```

You should see `CREATE ROLE`, `CREATE DATABASE`, `ALTER ROLE`, `CREATE EXTENSION`, and `ALTER SCHEMA`. If `CREATE EXTENSION` says the library is not preloaded, set `shared_preload_libraries=pg_stat_statements`, restart the pod, and create the extension again.

## 5. Cache

The service expects a Redis-compatible cache with authentication. The stand-in is Valkey from the c9s Redis image, for the same CPU reason as PostgreSQL. The service name is `valkey`. The operator secret key remains `redis-password`.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: valkey
  namespace: cost-deps
spec:
  selector:
    app: valkey
  ports:
    - name: valkey
      port: 6379
      targetPort: 6379
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: valkey
  namespace: cost-deps
spec:
  replicas: 1
  selector:
    matchLabels:
      app: valkey
  template:
    metadata:
      labels:
        app: valkey
    spec:
      containers:
        - name: valkey
          image: quay.io/sclorg/redis-7-c9s:c9s
          imagePullPolicy: IfNotPresent
          ports:
            - name: valkey
              containerPort: 6379
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: valkey-auth
                  key: password
          readinessProbe:
            exec:
              command: ["sh", "-c", "redis-cli -a \"$REDIS_PASSWORD\" ping | grep -q PONG"]
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            requests:
              cpu: 50m
              memory: 128Mi
            limits:
              memory: 512Mi
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
            runAsNonRoot: true
            seccompProfile:
              type: RuntimeDefault
EOF

oc rollout status deployment/valkey -n cost-deps
```

Success is `deployment "valkey" successfully rolled out`.

## 6. Kafka

The listener consumes `platform.upload.announce`. One KRaft broker is enough. `CLUSTER_ID` is fixed for the life of the log volume. Generate it once and save it. The service sets `publishNotReadyAddresses: true` so the broker can resolve its own advertised address before it is Ready. The image cannot write `/opt/kafka/config` or `/opt/kafka/logs`, so an init container copies the config onto an emptyDir and a second emptyDir is mounted at the log path. The data volume is the PVC at `/tmp/kafka-logs`.

```bash
export KAFKA_CLUSTER_ID="$(python3 -c 'import base64,uuid; print(base64.urlsafe_b64encode(uuid.uuid4().bytes).decode().rstrip("="))')"
echo "KAFKA_CLUSTER_ID=$KAFKA_CLUSTER_ID" >> ~/cost-mgmt-rebuild/secrets.env
echo "$KAFKA_CLUSTER_ID"
```

Apply in the same shell so `${KAFKA_CLUSTER_ID}` is set. This heredoc is unquoted on purpose.

```bash
set -a; source ~/cost-mgmt-rebuild/site.env; set +a

cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: cost-kafka
  namespace: cost-deps
spec:
  publishNotReadyAddresses: true
  selector:
    app: cost-kafka
  ports:
    - name: plaintext
      port: 9092
      targetPort: 9092
    - name: controller
      port: 9093
      targetPort: 9093
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cost-kafka
  namespace: cost-deps
spec:
  serviceName: cost-kafka
  replicas: 1
  selector:
    matchLabels:
      app: cost-kafka
  template:
    metadata:
      labels:
        app: cost-kafka
    spec:
      initContainers:
        - name: copy-config
          image: apache/kafka:3.9.1
          imagePullPolicy: IfNotPresent
          command: ["sh", "-c", "cp -R /opt/kafka/config/. /config-rw/ && chmod -R a+rX,g+w /config-rw || cp -R /opt/kafka/config/. /config-rw/"]
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
            runAsNonRoot: true
            seccompProfile:
              type: RuntimeDefault
          volumeMounts:
            - name: config
              mountPath: /config-rw
      containers:
        - name: kafka
          image: apache/kafka:3.9.1
          imagePullPolicy: IfNotPresent
          ports:
            - name: plaintext
              containerPort: 9092
            - name: controller
              containerPort: 9093
          env:
            - name: KAFKA_NODE_ID
              value: "1"
            - name: KAFKA_PROCESS_ROLES
              value: broker,controller
            - name: KAFKA_LISTENERS
              value: PLAINTEXT://:9092,CONTROLLER://:9093
            - name: KAFKA_ADVERTISED_LISTENERS
              value: PLAINTEXT://cost-kafka.cost-deps.svc.cluster.local:9092
            - name: KAFKA_CONTROLLER_LISTENER_NAMES
              value: CONTROLLER
            - name: KAFKA_LISTENER_SECURITY_PROTOCOL_MAP
              value: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
            - name: KAFKA_CONTROLLER_QUORUM_VOTERS
              value: 1@cost-kafka.cost-deps.svc.cluster.local:9093
            - name: KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR
              value: "1"
            - name: KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR
              value: "1"
            - name: KAFKA_TRANSACTION_STATE_LOG_MIN_ISR
              value: "1"
            - name: KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS
              value: "0"
            - name: KAFKA_NUM_PARTITIONS
              value: "3"
            - name: CLUSTER_ID
              value: "${KAFKA_CLUSTER_ID}"
            - name: KAFKA_LOG_DIRS
              value: /tmp/kafka-logs
          readinessProbe:
            tcpSocket:
              port: 9092
            initialDelaySeconds: 20
            periodSeconds: 10
          resources:
            requests:
              cpu: 200m
              memory: 1Gi
            limits:
              memory: 2Gi
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
            runAsNonRoot: true
            seccompProfile:
              type: RuntimeDefault
          volumeMounts:
            - name: data
              mountPath: /tmp/kafka-logs
            - name: config
              mountPath: /opt/kafka/config
            - name: klogs
              mountPath: /opt/kafka/logs
      volumes:
        - name: config
          emptyDir: {}
        - name: klogs
          emptyDir: {}
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: ${STORAGE_CLASS}
        resources:
          requests:
            storage: 20Gi
EOF

oc rollout status statefulset/cost-kafka -n cost-deps

oc exec -n cost-deps cost-kafka-0 -- printenv CLUSTER_ID
oc exec -n cost-deps cost-kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --if-not-exists \
  --topic platform.upload.announce \
  --partitions 3 --replication-factor 1
```

`CLUSTER_ID` must print a short string, not a blank line. The topic command should print `Created topic platform.upload.announce`.

**Lab example.** Cluster id `7A4vbOPATHuNrZ3E528OhA` on the rebuild. An earlier volume on the same cluster had used `MkU3OEVBNTcwNTJENDM2Qk`. Reusing a disk requires the original id.

## 7. Object storage

The custom resource is pointed at a service named `minio` on port 9000, bucket `koku-bucket`, path-style, region `us-east-1`. Any S3 API can sit behind that service. Quay MinIO images were not pullable with the lab cluster's pull secret, so the lab ran SeaweedFS and kept the Kubernetes service name `minio`.

If you already have S3, skip this section and set the endpoint, bucket, and keys in the custom resource and in `cost-object-storage`.

```bash
set -a; source ~/cost-mgmt-rebuild/site.env; set +a

cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-data
  namespace: cost-deps
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: ${STORAGE_CLASS}
  resources:
    requests:
      storage: 50Gi
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: cost-deps
spec:
  selector:
    app: minio
  ports:
    - name: s3
      port: 9000
      targetPort: 9000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: cost-deps
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - name: minio
          image: docker.io/chrislusf/seaweedfs:3.80
          imagePullPolicy: IfNotPresent
          command: ["sh", "-c"]
          args:
            - |
              set -eu
              umask 077
              printf '%s\n' "{\"identities\":[{\"name\":\"costadmin\",\"credentials\":[{\"accessKey\":\"\${S3_ACCESS_KEY}\",\"secretKey\":\"\${S3_SECRET_KEY}\"}],\"actions\":[\"Admin\",\"Read\",\"Write\",\"List\",\"Tagging\"]}]}" > /tmp/s3.json
              exec weed server -dir=/data -ip.bind=0.0.0.0 -master.port=9333 -volume.port=8080 -filer.port=8888 -s3 -s3.port=9000 -s3.config=/tmp/s3.json
          env:
            - name: S3_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: minio-auth
                  key: access-key
            - name: S3_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: minio-auth
                  key: secret-key
          ports:
            - name: s3
              containerPort: 9000
          readinessProbe:
            tcpSocket:
              port: 9000
            initialDelaySeconds: 8
            periodSeconds: 5
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              memory: 1Gi
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
            runAsNonRoot: true
            seccompProfile:
              type: RuntimeDefault
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: minio-data
EOF

oc rollout status deployment/minio -n cost-deps

oc exec -i -n cost-deps deploy/minio -- weed shell -master=localhost:9333 <<'EOF'
s3.bucket.create -name koku-bucket
s3.bucket.list
EOF
```

`oc exec` must include `-i`, or the shell never sees the create command and only prints the master connection. Success includes `created bucket koku-bucket` and a list line for that bucket.

The `\${S3_ACCESS_KEY}` escapes are so your shell does not substitute those variables while applying the manifest. The container expands them from the secret.

## 8. Keycloak

Use an existing realm. This guide does not deploy a second Keycloak. The lab realm was `kubernetes`. Unmanaged user attributes must be enabled or Keycloak drops `org_id` and `account_number`.

Sign in to `https://${KC_HOST}/admin`. If your Keycloak stores the bootstrap admin in `/etc/keycloak/keycloak.env` on the host, read it with a TTY so `sudo` can prompt. Do not paste the password into a ticket or a git repo.

```bash
ssh -t "admin@${KC_HOST}" 'sudo grep ^KC_BOOTSTRAP_ADMIN_ /etc/keycloak/keycloak.env'
```

Without `-t`, sudo fails with "a terminal is required to read the password".

### 8.1 Unmanaged attributes and the org-admin role

In Keycloak 26 the unmanaged-attributes control is not a dropdown on the attribute list. Open the realm, then **Realm settings → User profile → JSON editor**. Set:

```json
"unmanagedAttributePolicy": "ENABLED"
```

Save. Leave the `attributes` array in place.

Then **Realm roles**. Create `org-admin` if it is missing.

**Lab example.** Realm `kubernetes` on `https://kc01.dragon.internal`. `unmanagedAttributePolicy` was already `ENABLED` in the JSON. `org-admin` was already in the role list.

### 8.2 User costadmin

Search **Users** for `costadmin` before you create one. If the user exists, edit it.

- Email `costadmin@your-domain`, email verified on.
- Attributes `org_id` and `account_number`, both set to the org id you chose (`1000001` in the lab).
- A password with **Temporary** off. Store it without putting it on the command line:

```bash
read -s -p 'costadmin password: ' COSTADMIN_PASSWORD; echo
echo "COSTADMIN_PASSWORD=$COSTADMIN_PASSWORD" >> ~/cost-mgmt-rebuild/secrets.env
unset COSTADMIN_PASSWORD
```

- **Role mapping**: realm role `org-admin`. Use **Assign role → Realm roles**. The dialog title must name `costadmin`.

### 8.3 UI client

Create or update confidential client `cost-management-ui`.

- Standard flow on. Direct access grants, implicit flow, and service accounts off.
- Valid redirect URI `https://cost-onprem-ui-openshift-operators.${APPS_DOMAIN}/oauth2/callback`
- Web origin `https://cost-onprem-ui-openshift-operators.${APPS_DOMAIN}`

The host only exists after the custom resource is created in `openshift-operators`. It is predictable, so you can set the redirect now. If you later put the custom resource in another namespace, the route host changes and login loops until this redirect matches.

Open **Client scopes** and click `cost-management-ui-dedicated`. Add these mappers if they are missing.

| Mapper | Type | Required settings |
|---|---|---|
| `aud-cost-management-operator` | Audience | Included client audience and custom audience both `cost-management-operator`. Add to access token on. |
| `aud-cost-management-ui` | Audience | Custom audience `cost-management-ui`. Add to access token on. |
| `org_id` | User Attribute | Attribute and claim `org_id`, type String, not multivalued. |
| `account_number` | User Attribute | Attribute and claim `account_number`, type String, not multivalued. |

On the two user-attribute mappers, turn all of these on:

- Add to access token
- Add to lightweight access token
- Add to ID token
- Add to userinfo

Keycloak 26 omits the claims from the access token when "Add to lightweight access token" is off. The gateway then returns `401 Unauthorized: Missing org_id in JWT claims`. The UI treats every 401 as a dead session and sends the browser to `/logout`, which looks like the page loads and signs you out. A 403 does not log you out.

Copy the client secret from the **Credentials** tab and store it. The OpenShift secret uses hyphenated keys.

```bash
read -s -p 'UI client secret: ' UI_CLIENT_SECRET; echo
echo "UI_CLIENT_SECRET=$UI_CLIENT_SECRET" >> ~/cost-mgmt-rebuild/secrets.env
set -a; source ~/cost-mgmt-rebuild/secrets.env; set +a

oc create secret generic cost-onprem-ui-oauth-client -n openshift-operators \
  --from-literal=client-id=cost-management-ui \
  --from-literal=client-secret="$UI_CLIENT_SECRET"
unset UI_CLIENT_SECRET
```

### 8.4 Metrics client

The client id must be the UUID in `METRICS_CLIENT_UUID`. insights-rbac only accepts service-account usernames of the form `service-account-<uuid>`. A client id of `cost-management-metrics-operator` produces a username the API rejects with 403.

```bash
source ~/cost-mgmt-rebuild/secrets.env
echo "$METRICS_CLIENT_UUID"
```

Create a confidential client whose client id is that UUID.

- Client authentication on. Standard flow off. Service accounts on.
- The same four mappers as the UI client, including lightweight access token on the two attribute mappers.
- Client scope `api.console`, included in the token, added to this client as a **Default** scope. The operator requests `scope=api.console`. Without the scope the token call returns `400 invalid_scope`. Create the scope if it does not exist.

Keycloak hides service-account users from the Users list. Open **Clients → your UUID → Service account roles** and click the username `service-account-<uuid>`.

On that user:

- Attributes `org_id` and `account_number` set to your org id.
- Realm role `org-admin`, via **Assign role → Realm roles**. The dialog title must be the service-account username. If it says `costadmin`, cancel and open the service-account user again.

Store the client secret. This secret uses underscored keys and lives in `cost-onprem`.

```bash
read -s -p 'Metrics client secret: ' METRICS_CLIENT_SECRET; echo
echo "METRICS_CLIENT_SECRET=$METRICS_CLIENT_SECRET" >> ~/cost-mgmt-rebuild/secrets.env
set -a; source ~/cost-mgmt-rebuild/secrets.env; set +a

oc create secret generic service-account-auth-secret -n cost-onprem \
  --from-literal=client_id="$METRICS_CLIENT_UUID" \
  --from-literal=client_secret="$METRICS_CLIENT_SECRET"
unset METRICS_CLIENT_SECRET
```

**Lab example.** Metrics client id `8e16695b-f9fc-45ab-bc45-ebe06d5dcaf9`. An extra Audience mapper named `cost-management-ui` was left in place beside `aud-cost-management-ui`. It did not need to be deleted.

### 8.5 Identity provider CA

The service verifies Keycloak's certificate with a CA you provide. Keycloak often sends only its leaf certificate, so saving the handshake with `openssl s_client` does not yield the CA. Download the CA from your identity management server and confirm the subject before you create the secret.

```bash
set -a; source ~/cost-mgmt-rebuild/site.env; set +a
curl -fsSk "$CA_URL" -o ~/cost-mgmt-rebuild/ipa-ca.crt
openssl x509 -in ~/cost-mgmt-rebuild/ipa-ca.crt -noout -subject

oc create secret generic keycloak-ca -n openshift-operators \
  --from-file=ca.crt="$HOME/cost-mgmt-rebuild/ipa-ca.crt"
```

**Lab example.** Subject `O=DRAGON.INTERNAL, CN=Certificate Authority`, from `https://idm01.dragon.internal/ipa/config/ca.crt`. A script that kept writing after `END CERTIFICATE` produced a file OpenSSL could not decode. Do not use that approach.

## 9. Cluster DNS

Pods must resolve the Keycloak hostname. Forward only that name to your internal DNS. Forwarding the whole parent zone steals the cluster's own `*.apps` names.

```bash
set -a; source ~/cost-mgmt-rebuild/site.env; set +a

oc patch dns.operator.openshift.io/default --type=merge -p "{
  \"spec\": {
    \"servers\": [{
      \"name\": \"keycloak\",
      \"zones\": [\"${KC_HOST}\"],
      \"forwardPlugin\": {
        \"policy\": \"Random\",
        \"upstreams\": [\"${DNS_UPSTREAM}\"]
      }
    }]
  }
}"

oc delete pod -n openshift-dns -l dns.operator.openshift.io/daemonset-dns=default

oc run -n cost-deps dnscheck --rm -it --restart=Never \
  --image=registry.redhat.io/ubi9/ubi-minimal -- \
  getent hosts "${KC_HOST}"
```

A PodSecurity warning on that temporary pod is expected. The line you want is the Keycloak address. The pod is deleted when the command finishes.

Your workstation also needs the UI hostname if laptop DNS does not know `*.apps`. Use the node address that reaches the cluster ingress.

```bash
grep cost-onprem-ui-openshift-operators /etc/hosts || true
```

Add a line only when it is missing. Example shape:

```text
<ingress-or-node-ip>   cost-onprem-ui-openshift-operators.<APPS_DOMAIN>
```

The browser stays on the UI host. The UI proxies `/api` to the gateway, so the gateway hostname does not need a workstation hosts entry.

**Lab example.** Forward `kc01.dragon.internal` to `192.168.1.10`. `getent` returned `192.168.1.13`. The laptop hosts line was `192.168.1.238  cost-onprem-ui-openshift-operators.apps.sno.dragon.internal`.

## 10. Cost Management Service Operator

This operator is not in `redhat-operators`. A CatalogSource pulls `quay.io/project-koku/koku-service-operator-catalog:v0.0.1`. The console may say "provided by Red Hat" because that string is the catalog's publisher field. It does not mean the operator is in the default catalog.

Subscribe in `openshift-operators`. A subscription in its own namespace fails the install mode check.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: koku-service-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: Cost Management
  image: quay.io/project-koku/koku-service-operator-catalog:v0.0.1
  publisher: Red Hat
  sourceType: grpc
  updateStrategy:
    registryPoll:
      interval: 30m
EOF

oc wait --for=condition=Ready pods -l olm.catalogSource=koku-service-operator-catalog \
  -n openshift-marketplace --timeout=300s

cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: koku-service-operator
  namespace: openshift-operators
spec:
  channel: beta
  installPlanApproval: Automatic
  name: koku-service-operator
  source: koku-service-operator-catalog
  sourceNamespace: openshift-marketplace
EOF

until oc get csv -n openshift-operators koku-service-operator.v0.0.1 \
  -o jsonpath='{.status.phase}' 2>/dev/null | grep -q Succeeded
do
  sleep 5
done

oc get csv -n openshift-operators koku-service-operator.v0.0.1
oc get deploy -n openshift-operators koku-service-operator-controller-manager
```

The CSV phase should be `Succeeded` and the controller deployment `1/1`. The sample RBAC image tag `73870d8` is not published. The custom resource later pins `quay.io/project-koku/insights-rbac:34e25ed`.

## 11. UI image

Every published `quay.io/insights-onprem/koku-ui-onprem` tag through September 2026 is UBI 10. On a CPU without AVX2, nginx dies before it listens. The operator can still mark the UI ready, because that check only means the OAuth secret exists. Confirm the pod is `2/2`.

If your CPU has AVX2, skip this section and set the UI tag in the next section to a published tag instead of `lab-c9s`.

`podman create` copies the static files without running the UBI 10 entrypoint. The rebuild is those files on `ubi9/nginx-126`, which listens on 8080 and includes `/opt/app-root/etc/nginx.default.d/*.conf`. That is where the operator mounts its nginx config.

```bash
podman create --name koku-ui-src quay.io/insights-onprem/koku-ui-onprem:latest
mkdir -p /tmp/koku-ui-build/src
podman cp koku-ui-src:/opt/app-root/src/. /tmp/koku-ui-build/src/
podman rm koku-ui-src

cat > /tmp/koku-ui-build/Containerfile <<'EOF'
FROM registry.access.redhat.com/ubi9/nginx-126:latest
USER 0
RUN rm -rf /opt/app-root/src && mkdir -p /opt/app-root/src
COPY --chown=1001:0 src/ /opt/app-root/src/
USER 1001
EOF

podman build -t quay.io/insights-onprem/koku-ui-onprem:lab-c9s /tmp/koku-ui-build
podman save quay.io/insights-onprem/koku-ui-onprem:lab-c9s | gzip > /tmp/koku-ui-lab-c9s.tar.gz
ls -lh /tmp/koku-ui-lab-c9s.tar.gz
```

The cluster image registry on the lab was `Removed`, so there is nowhere in-cluster to push. Load the archive onto the node. In one terminal, leave this running and wait for `Starting pod/...`:

```bash
set -a; source ~/cost-mgmt-rebuild/site.env; set +a
oc debug "node/${NODE_NAME}" -n default --preserve-pod -- sleep 900
```

In a second terminal, do not invent the pod name. Look it up:

```bash
POD=$(oc get pods -n default -o name | sed -n 's|^pod/||p' | grep debug | head -1)
echo "using $POD"

oc cp /tmp/koku-ui-lab-c9s.tar.gz "default/${POD}:/host/tmp/koku-ui-lab-c9s.tar.gz"
oc exec -n default "$POD" -- chroot /host podman load -i /tmp/koku-ui-lab-c9s.tar.gz
oc exec -n default "$POD" -- chroot /host podman image exists quay.io/insights-onprem/koku-ui-onprem:lab-c9s && echo IMAGE_OK
oc exec -n default "$POD" -- rm -f /host/tmp/koku-ui-lab-c9s.tar.gz
oc delete pod -n default "$POD"
```

`using` must not be a placeholder. You want `Loaded image: quay.io/insights-onprem/koku-ui-onprem:lab-c9s` and `IMAGE_OK`. The custom resource uses `imagePullPolicy: IfNotPresent`, so kubelet uses this local image. If the node loses its container store, load the archive again.

**Lab example.** The archive was 134M. The debug pod was `00-0a-f7-a6-59-d8-debug-n6fwv`. An earlier attempt failed because the shell still had `POD` set to the example suffix `xxxxx`.

## 12. CostManagementServiceConfig

Apply this only after the secrets, dependency services, Keycloak redirect, and the UI image exist. The namespace is `openshift-operators`. A copy in `cost-onprem` is ignored.

`issuerURL` must include `/realms/<realm>`. `ros.enabled: false` skips the resource-optimization images. `deploy: false` on the database and cache tells the operator to use the services you already created.

```bash
set -a; source ~/cost-mgmt-rebuild/site.env; set +a

cat <<EOF | oc apply -f -
apiVersion: service.costmanagement.openshift.io/v1alpha1
kind: CostManagementServiceConfig
metadata:
  name: cost-onprem
  namespace: openshift-operators
spec:
  profile: standard
  global:
    clusterDomain: ${APPS_DOMAIN}
    pullPolicy: IfNotPresent
  database:
    deploy: false
    host: postgresql.cost-deps.svc
    port: 5432
    sslMode: disable
    secretName: cost-db-credentials
  cache:
    deploy: false
    host: valkey.cost-deps.svc
    port: 6379
    auth:
      enabled: true
      secretName: cost-cache-credentials
  kafka:
    bootstrapServers: cost-kafka.cost-deps.svc.cluster.local:9092
    securityProtocol: PLAINTEXT
  objectStorage:
    endpoint: minio.cost-deps.svc
    port: 9000
    useSSL: false
    secretName: cost-object-storage
    buckets:
      koku: koku-bucket
    s3:
      addressingStyle: path
      region: us-east-1
  auth:
    keycloak:
      url: https://${KC_HOST}
      realm: ${KC_REALM}
      issuerURL: https://${KC_HOST}/realms/${KC_REALM}
      audiences:
        - cost-management-operator
        - cost-management-ui
      tls:
        caCertSecretName: keycloak-ca
    envoy:
      replicas: 1
      image:
        repository: registry.redhat.io/openshift-service-mesh/proxyv2-rhel9
        tag: "2.6"
  ingress:
    image:
      repository: quay.io/iop/ingress
      tag: sha-6dde23d
    validTypes: hccm
    maxUploadSize: 104857600
  rbac:
    image:
      repository: quay.io/project-koku/insights-rbac
      tag: 34e25ed
  costManagement:
    dataRetentionMonths: 4
    api:
      replicas: 1
      image:
        repository: quay.io/project-koku/koku
        tag: 422f758
    masu:
      replicas: 1
      image:
        repository: quay.io/project-koku/koku
        tag: 422f758
    listener:
      enabled: true
      replicas: 1
  ros:
    enabled: false
  ui:
    replicaCount: 1
    app:
      image:
        repository: quay.io/insights-onprem/koku-ui-onprem
        tag: lab-c9s
        pullPolicy: IfNotPresent
    oauthProxy:
      cookieRefresh: 4m
      cookieExpire: 720h
      image:
        repository: registry.redhat.io/rhceph/oauth2-proxy-rhel9
        tag: v7.6.0
  gatewayRoute:
    tls:
      termination: edge
      insecureEdgeTerminationPolicy: Redirect
EOF

oc get cmsc cost-onprem -n openshift-operators
oc get pods -n openshift-operators -l app.kubernetes.io/component=ui
curl -skI "https://cost-onprem-ui-openshift-operators.${APPS_DOMAIN}" | head -n 15
```

Wait until the phase is `Ready` and the UI pod is `2/2`. The first `oc get` can still say `Progressing` for several minutes while images pull. Re-run it. `curl` should be `302` to your Keycloak authorize URL with `client_id=cost-management-ui` and the callback host above.

Image tags above are the ones that ran with operator `0.0.1` on 2 October 2026. If a later catalog documents different tags, prefer those, except do not use RBAC tag `73870d8`.

**Lab example.** Phase reached `Ready` about ten minutes after create. UI pod `cost-onprem-ui-55d76bd9d8-9t9hb` was `2/2`. A short-lived mount warning for secret `cost-onprem-ui-tls` cleared on its own.

## 13. Metrics operator

Install this operator in `cost-onprem` with an OperatorGroup whose target is only that namespace. Subscribing it in `openshift-operators` fails because AllNamespaces is not a supported install mode. Do not apply the metrics custom resource yet. The tenant in the next section has to exist first, or the first upload is discarded.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cost-onprem
  namespace: cost-onprem
spec:
  targetNamespaces:
    - cost-onprem
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: costmanagement-metrics-operator
  namespace: cost-onprem
spec:
  channel: stable
  installPlanApproval: Automatic
  name: costmanagement-metrics-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

until oc get csv -n cost-onprem \
  -l operators.coreos.com/costmanagement-metrics-operator.cost-onprem= \
  -o jsonpath='{.items[0].status.phase}' 2>/dev/null | grep -q Succeeded
do
  sleep 5
done

oc get csv -n cost-onprem
```

You should see `costmanagement-metrics-operator` at phase `Succeeded`. A copy of the service operator CSV in this namespace is expected, because that operator is installed for all namespaces.

**Lab example.** Version `4.4.2`, replacing `4.4.1`.

## 14. Tenant and metrics service-account access

Koku creates the customer schema on a non-GET. The metrics operator's first call is a GET, so the org never appears and sources return 403, "Tenant does not exist", until you create the row.

`ENHANCED_ORG_ADMIN` is hardcoded false on the API deployment. An `org-admin` token does not bypass RBAC, and service accounts are never treated as org admins. The metrics principal has to be in the group `Default admin access`, which carries `cost-management:*:*`.

Starting Django in the RBAC pod also runs the seed. It logs permissions and roles as "eligible for removal". On this build that log is not a delete. `Cost Administrator` remains.

```bash
set -a
source ~/cost-mgmt-rebuild/site.env
source ~/cost-mgmt-rebuild/secrets.env
set +a

oc exec -i -n openshift-operators deploy/cost-onprem-koku-api -c koku-api -- \
  /opt/koku/.venv/bin/python - <<PY
import os
os.chdir("/opt/koku/koku")
os.environ["DJANGO_SETTINGS_MODULE"] = "koku.settings"
os.environ["PYTHONPATH"] = "/opt/koku/koku"
import django
django.setup()
from api.iam.models import Customer
obj, created = Customer.objects.get_or_create(
    org_id="${ORG_ID}",
    defaults={"account_id": "${ACCOUNT_NUMBER}", "schema_name": "org${ORG_ID}"},
)
print(created, obj.schema_name)
PY

oc exec -i -n openshift-operators deploy/cost-onprem-rbac-api -c rbac-api -- \
  /opt/rbac/.venv/bin/python - <<PY
import os
os.chdir("/opt/rbac/rbac")
os.environ["DJANGO_SETTINGS_MODULE"] = "rbac.settings"
os.environ["PYTHONPATH"] = "/opt/rbac/rbac"
import django
django.setup()
from api.models import Tenant
from management.models import Group, Principal
username = "service-account-${METRICS_CLIENT_UUID}"
tenant, _ = Tenant.objects.get_or_create(
    org_id="${ORG_ID}",
    defaults={"tenant_name": "org${ORG_ID}"},
)
principal, _ = Principal.objects.get_or_create(
    username=username,
    tenant=tenant,
    defaults={"type": "service-account"},
)
group = Group.objects.get(name="Default admin access", admin_default=True)
principal.group.add(group)
print("granted", username, "->", group.name)
PY
```

You want a line `True org<id>` or `False org<id>`, and `granted service-account-<uuid> -> Default admin access`.

**Lab example.** `True org1000001` and `granted service-account-8e16695b-f9fc-45ab-bc45-ebe06d5dcaf9 -> Default admin access`.

## 15. Metrics custom resource

This object tells the metrics operator where to upload and how to authenticate. `create_source: true` is required. The knowledgebase sample says `false`, but nothing else creates the integration, and uploads wait while `source_defined` is false.

`validate_cert: false` skips verification of the identity-provider certificate on the token call and the upload. This custom resource has no CA field. The alternative is to trust that CA cluster-wide, which reboots a single-node cluster.

`upload_cycle: 5` is stored and then raised to 60 (`COST-4528`). The first upload runs immediately because there is no previous success. Later uploads are about an hour apart. `disable_metrics_collection_resource_optimization: true` matches `ros.enabled: false`.

The operator creates PVC `costmanagement-metrics-operator-data`, 10Gi, on your default or the storage class it selects.

```bash
set -a; source ~/cost-mgmt-rebuild/site.env; set +a

cat <<EOF | oc apply -f -
apiVersion: costmanagement-metrics-cfg.openshift.io/v1beta1
kind: CostManagementMetricsConfig
metadata:
  name: costmanagementmetricscfg-sample
  namespace: cost-onprem
spec:
  api_url: https://cost-onprem-gateway-openshift-operators.${APPS_DOMAIN}
  authentication:
    type: service-account
    secret_name: service-account-auth-secret
    token_url: https://${KC_HOST}/realms/${KC_REALM}/protocol/openid-connect/token
  packaging:
    max_reports_to_store: 30
    max_size_MB: 100
  prometheus_config:
    service_address: https://thanos-querier.openshift-monitoring.svc:9091
    skip_tls_verification: false
    context_timeout: 120
    collect_previous_data: true
    disable_metrics_collection_cost_management: false
    disable_metrics_collection_resource_optimization: true
  source:
    name: ${SOURCE_NAME}
    sources_path: /api/cost-management/v1/
    check_cycle: 1440
    create_source: true
  upload:
    ingress_path: /api/ingress/v1/upload
    upload_cycle: 5
    upload_toggle: true
    validate_cert: false
EOF
```

`source.name` is `SOURCE_NAME` from `site.env`. That string is the integration name in the UI. The lab used `sno.dragon.internal`.

Status is empty until the operator finishes the first pass. Wait a few minutes, then:

```bash
oc get costmanagementmetricsconfig -n cost-onprem costmanagementmetricscfg-sample \
  -o jsonpath='source_defined={.status.source.source_defined} upload={.status.upload.last_upload_status} at={.status.upload.last_successful_upload_time} prom={.status.prometheus.prometheus_connected} cycle={.status.upload.upload_cycle}{"\n"}'
```

A good line is `source_defined=true`, `upload=202 Accepted`, `prom=true`, `cycle=60`.

The first payload can still be dropped. The listener logs `Received unexpected OCP report` when the source row is not visible at ingest time, even if a `POST /api/cost-management/v1/sources` returns 201 a few seconds earlier. That payload is deleted. The next cycle, about 60 minutes later, is kept. Do not wait 24 hours. The "up to 24 hours" text in the UI is generic copy for an integration that has no processed report yet.

Confirm the cluster id the operator recorded:

```bash
oc get clusterversion version -o jsonpath='{.spec.clusterID}{"\n"}'
```

The provider credential must be exactly `{"cluster_id":"<that id>"}`.

**Lab example.** First status query at 90 seconds was blank. By 07:25 UTC the upload was `202 Accepted`, Prometheus was connected, and `POST /sources` returned 201. The listener then logged `Received unexpected OCP report from 9c882036-e1c7-4452-8a75-5b07ebb22b2a` for that first payload.

## 16. Log in and grant costadmin

Open `https://cost-onprem-ui-openshift-operators.${APPS_DOMAIN}` and sign in as `costadmin`. Use a fresh browser session after any mapper change so the access token contains `org_id`.

A new human user lands in `Default access`. That group can read OpenShift usage. It cannot open Settings, price lists, or cost models. The UI shows "You do not have access to Cost management" on Settings. That is a 403, not a logout.

After you have signed in once, grant `costadmin` the same admin group and restart the API. The restart drops a cached 403. Refresh the page. Do not expect the old tab to notice the new role by itself.

```bash
set -a; source ~/cost-mgmt-rebuild/site.env; set +a

oc exec -i -n openshift-operators deploy/cost-onprem-rbac-api -c rbac-api -- \
  /opt/rbac/.venv/bin/python - <<PY
import os
os.chdir("/opt/rbac/rbac")
os.environ["DJANGO_SETTINGS_MODULE"] = "rbac.settings"
os.environ["PYTHONPATH"] = "/opt/rbac/rbac"
import django
django.setup()
from api.models import Tenant
from management.models import Group, Principal
tenant = Tenant.objects.get(org_id="${ORG_ID}")
principal, _ = Principal.objects.get_or_create(
    username="costadmin",
    tenant=tenant,
    defaults={"type": "user"},
)
group = Group.objects.get(name="Default admin access", admin_default=True)
principal.group.add(group)
print("granted costadmin ->", group.name)
PY

oc rollout restart deploy/cost-onprem-koku-api -n openshift-operators
oc rollout status deploy/cost-onprem-koku-api -n openshift-operators
```

You want `granted costadmin -> Default admin access` and a successful rollout. Refresh Settings.

**Lab example.** Settings showed the lock page until this grant. After the API rollout and a refresh, Settings opened.

## 17. Price list and cost model

Usage data can be present while every dollar total is still zero. A price list holds rates. A cost model attaches one price list to one cluster. A cluster belongs to only one cost model. Markup changes raw infrastructure cost. It does not change price-list rates. The price-list currency cannot be changed after you create the list, and the validity range has to cover the month you want priced.

**Add rate** stays disabled until Name is set. Keep four decimal places on small rates. `0.00` drops the rate.

Supplementary is the calculation type for CPU, memory, and storage. Infrastructure is for a flat fee such as a cluster-hour. Usage prices what the workload consumed. Request prices what it reserved. Adding both for the same metric bills the same cores twice. Adding a node-month on top of CPU and memory also bills the machine twice.

Cost distribution divides shared cost onto projects. Platform cost is OpenShift's own projects, such as monitoring and ingress. Worker unallocated is node capacity no pod requested or used. With distribution on, each project is charged its own usage plus a share of that overhead. With it off, those amounts stay on their own lines.

The lab entered an example list and model after the first kept data appeared. Recalculate rates for your currency and date. These numbers are not a Red Hat price.

**Lab example, price list `Lab rates`.** Currency AUD. Validity 1 September 2026 through 31 December 2026. Rates were derived from AWS Sydney on-demand Linux on 29 September 2026 (`c6i.xlarge` and `r6i.xlarge` for CPU and memory, gp3 for storage), converted at about A\$1.43 per US dollar.

| Name | Metric | Measurement | Calculation | Rate |
|---|---|---|---|---|
| CPU usage | CPU | Usage (core-hours) | Supplementary | 0.07 AUD |
| Memory usage | Memory | Usage (GiB-hours) | Supplementary | 0.0048 AUD |
| Storage usage | Storage | Request (GiB-month) | Supplementary | 0.14 AUD |

**Lab example, cost model `SNO`.** Source type OpenShift Container Platform, currency AUD, markup 0. Price list `Lab rates` at priority 1. Distribution on, split by CPU, with platform, worker unallocated, network unattributed, storage unattributed, and GPU unallocated all included. Integration `sno.dragon.internal`.

Open OpenShift details for the current month and refresh after processing. Supplementary cost moves off zero. Infrastructure stays zero until you add an infrastructure rate.

## Troubleshooting

| What you see | What it means | What to do |
|---|---|---|
| `oc login` says it received an HTML page | The URL is the console on port 443 | Use `https://api.<cluster>:6443` |
| Service CSV `Failed` in its own namespace | Install mode is AllNamespaces only | Subscribe in `openshift-operators` |
| Metrics CSV `Failed` in `openshift-operators` | AllNamespaces is not supported | OperatorGroup and subscription in `cost-onprem` |
| Custom resource never becomes Ready | The controller only watches its own namespace | Put the CR and its secrets in `openshift-operators` |
| Pods die with `CPU does not support x86-64-v3` | The image is RHEL 10 / UBI 10 | c9s database and cache images, and the UBI 9 UI rebuild |
| `permission denied for pg_stat_statements` | `koku` is not a superuser, or the extension is missing | Section 4 SQL |
| Kafka never Ready | It cannot resolve itself, or config is read-only | `publishNotReadyAddresses`, config init container, emptyDir for logs |
| Token `400 invalid_scope` | `api.console` is missing | Default client scope on the metrics client |
| Sources 403, invalid service-account username | Client id is not a UUID | Recreate the metrics client with `uuidgen` |
| Sources 403, tenant does not exist | The customer row is created only on a non-GET | Section 14 |
| Sources still 403 after the tenant exists | The service account is not in `Default admin access` | Section 14, then restart `cost-onprem-koku-api` |
| Settings says you do not have access | `costadmin` is only in `Default access` | Section 16, restart the API, refresh |
| UI loads, then returns to Keycloak | `org_id` is missing from the access token | Lightweight access token on the UI mappers, then a new login |
| OpenShift details stays Incomplete | The upload was accepted and then discarded | Confirm the source exists, wait for the next 60-minute upload |
| Status `upload_cycle` is 60 | Operator minimum | Leave it. The first upload is soon; later ones are hourly |
| Dollar totals stay at 0 | No cost model, or the price-list dates miss this month | Section 17 |

## Differences from the knowledgebase sample

These are the places this procedure does not follow the sample literally, and why.

1. The service custom resource and its secrets live in `openshift-operators`, because that is the namespace the controller watches.
2. Metrics `create_source` is `true`.
3. The metrics client id is a UUID. The UI OAuth secret uses hyphenated keys. The metrics secret uses underscored keys.
4. On Keycloak 26, `org_id` and `account_number` need the lightweight access token claim or the UI logs you out.
5. The Koku customer and both `Default admin access` grants are created by hand. `costadmin` is granted after the user exists, then `cost-onprem-koku-api` is restarted.
6. On CPUs without AVX2, PostgreSQL and the cache use c9s images and the UI is a local UBI 9 rebuild tagged `lab-c9s`.
7. Object storage in the lab is SeaweedFS behind a service named `minio`.
8. The RBAC image tag is `34e25ed`.
9. Metrics `validate_cert` is `false` so a single-node cluster does not need a new cluster-wide CA.
10. `upload_cycle` below 60 is stored as 60.
