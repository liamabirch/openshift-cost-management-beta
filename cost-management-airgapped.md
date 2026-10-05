# Cost Management self-managed — disconnected install

Use this guide when the OpenShift cluster cannot pull images from the internet. You copy the images onto media, load them into a registry the cluster can reach, then follow [cost-management-self-managed.md](./cost-management-self-managed.md) for namespaces, dependencies, Keycloak, the custom resources, the tenant, and the admin grant.

The product article is [Knowledgebase 7148128](https://access.redhat.com/articles/7148128). The operator source repository is [project-koku/koku-server-operator](https://github.com/project-koku/koku-server-operator). That repository is not the install path.

The connected guide is the procedure for your own cluster. Hostnames, storage classes, and CPU workarounds from the lab that proved it are labeled **Lab example** there and here. Replace `registry.example.com:8443` with your mirror.

## What is different from a connected cluster

On a connected cluster, Operator Lifecycle Manager pulls the catalog and the operands as it reconciles. On a disconnected cluster those pulls fail, so you mirror first.

Two things do not change:

- The service operator catalog is still `quay.io/project-koku/koku-service-operator-catalog:v0.0.1`. It is not in `redhat-operators`.
- The custom resource with `deploy: false` still expects PostgreSQL 16, a Redis-compatible cache, Kafka, an S3 API, and an OpenID Connect provider that the pods can reach. Mirroring does not create those services. They have to exist inside the disconnected environment, either as the stand-ins in the connected guide or as services you already run.

Cloud cost sources (AWS, Azure, GCP) are unsupported in this beta. Resource optimization stays off.

## What to mirror

| Operator | Package | Catalog you mirror |
|---|---|---|
| Cost Management Service Operator `0.0.1` | `koku-service-operator`, channel `beta` | `quay.io/project-koku/koku-service-operator-catalog:v0.0.1` |
| Cost Management Metrics Operator (verified at `4.4.2`) | `costmanagement-metrics-operator`, channel `stable` | `registry.redhat.io/redhat/redhat-operator-index:v4.22` |

Also mirror the operand images the custom resource pins, and the dependency images if you will run the stand-ins from the connected guide. Tags below are the ones that worked with operator `0.0.1` on 2 October 2026.

```text
quay.io/project-koku/koku-service-operator-catalog:v0.0.1
quay.io/project-koku/koku-service-operator:v0.0.1
quay.io/project-koku/koku:422f758
quay.io/project-koku/insights-rbac:34e25ed
quay.io/iop/ingress:sha-6dde23d
registry.redhat.io/openshift-service-mesh/proxyv2-rhel9:2.6
registry.redhat.io/rhceph/oauth2-proxy-rhel9:v7.6.0
registry.access.redhat.com/ubi9/ubi-minimal:9.7
quay.io/sclorg/postgresql-16-c9s:c9s
quay.io/sclorg/redis-7-c9s:c9s
docker.io/apache/kafka:3.9.1
docker.io/chrislusf/seaweedfs:3.80
```

Skip the PostgreSQL, Redis, Kafka, and SeaweedFS lines when those services already exist inside the environment. Do not mirror the whole platform release or the whole `redhat-operators` index for this install. Mirror the metrics package and the additional images above.

The UI image depends on the CPU. Nodes with AVX2 can mirror a published `quay.io/insights-onprem/koku-ui-onprem` tag and set that tag on the custom resource. Nodes without AVX2 cannot run those tags. Build the UBI 9 image in section 1 and carry the tarball. Do not put the local `lab-c9s` tag in the ImageSetConfiguration unless you have already pushed it to a registry oc-mirror can pull.

**Lab example.** The cluster CPU was a Xeon E5-2630 v2 without AVX2, so the UI was rebuilt as `lab-c9s` and loaded onto the single node. Object storage was SeaweedFS because the Quay MinIO image was not authorized. The metrics operator digest that ran was `registry.redhat.io/costmanagement/costmanagement-metrics-rhel9-operator@sha256:b95c040d61783987694a10649bd996d1ad7c151244866226a05b70e8642e2f95`. Mirroring the package pulls the digest the index currently points at. You do not have to list that digest twice.

## Prerequisites

You need a connected bastion with `oc` and the oc-mirror plugin v2 (`oc mirror --v2`), a mirror registry the disconnected cluster can pull from, and a pull secret that can read `registry.redhat.io` and `quay.io`. Add `docker.io` if you mirror Kafka or SeaweedFS. Save that file as `$XDG_RUNTIME_DIR/containers/auth.json`. Add the mirror registry credentials to the same file and to the cluster secret `pull-secret` in `openshift-config`.

The disconnected cluster in this procedure is OpenShift 4.22, which is the release the knowledgebase article was prepared for. If your minor version differs, use the matching `redhat-operator-index` tag.

Red Hat's procedure for the plugin is [Mirroring images for a disconnected installation using the oc-mirror plugin v2](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/disconnected_environments/about-installing-oc-mirror-v2).

## Order

1. Build the UI image on a connected host when the nodes lack x86-64-v3.
2. Write the ImageSetConfiguration.
3. Mirror to disk on the connected bastion.
4. Move the archives, and the UI tarball if you built one, into the disconnected environment.
5. Mirror from disk into the registry.
6. Apply the generated ImageDigestMirrorSet, ImageTagMirrorSet, and CatalogSource.
7. Add a CatalogSource for the service operator catalog.
8. Continue with the connected guide.

---

## 1. UI image on a connected host

This step is only for CPUs without AVX2. Published UI tags are UBI 10. nginx exits before it listens, while the operator can still report the UI component ready because that check only looks for the OAuth secret.

`podman create` copies the static files without running the UBI 10 entrypoint. The Containerfile puts those files on `ubi9/nginx-126`, which listens on 8080 and includes `/opt/app-root/etc/nginx.default.d/*.conf`. The operator mounts its nginx config there.

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
```

Carry `/tmp/koku-ui-lab-c9s.tar.gz` with the oc-mirror archives. After the mirror registry is reachable, push the image so the cluster can pull it:

```bash
podman load -i koku-ui-lab-c9s.tar.gz
podman tag quay.io/insights-onprem/koku-ui-onprem:lab-c9s \
  registry.example.com:8443/insights-onprem/koku-ui-onprem:lab-c9s
podman push registry.example.com:8443/insights-onprem/koku-ui-onprem:lab-c9s
```

If the cluster has no registry path that ImageTagMirrorSet will rewrite, load the tarball onto each node with `podman load` and set `imagePullPolicy: IfNotPresent` on the UI, as the connected guide does for a single node.

**Lab example.** The archive was 134M and was loaded onto node `00-0a-f7-a6-59-d8` through a debug pod. The cluster image registry was `Removed`, so there was nowhere in-cluster to push.

## 2. ImageSetConfiguration

`additionalImages` entries need an explicit registry hostname. `apache/kafka:3.9.1` is pulled as `docker.io/apache/kafka:3.9.1`. The metrics operator image comes from the package entry. Listing it again is optional.

oc-mirror mirrors the package's default channel even when you only ask for `stable`. Confirm the package exists in the index before a long mirror:

```bash
oc mirror list operators \
  --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.22 \
  --package=costmanagement-metrics-operator
```

```bash
mkdir -p ~/cost-mgmt-mirror && cd ~/cost-mgmt-mirror

cat > imageset-config-cost-mgmt.yaml <<'EOF'
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v2alpha1
mirror:
  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.22
      packages:
        - name: costmanagement-metrics-operator
          channels:
            - name: stable
              minVersion: "4.4.2"
              maxVersion: "4.4.2"
  additionalImages:
    - name: quay.io/project-koku/koku-service-operator-catalog:v0.0.1
    - name: quay.io/project-koku/koku-service-operator:v0.0.1
    - name: quay.io/project-koku/koku:422f758
    - name: quay.io/project-koku/insights-rbac:34e25ed
    - name: quay.io/iop/ingress:sha-6dde23d
    - name: registry.redhat.io/openshift-service-mesh/proxyv2-rhel9:2.6
    - name: registry.redhat.io/rhceph/oauth2-proxy-rhel9:v7.6.0
    - name: registry.access.redhat.com/ubi9/ubi-minimal:9.7
    - name: registry.access.redhat.com/ubi9/nginx-126:latest
    - name: quay.io/insights-onprem/koku-ui-onprem:latest
    - name: quay.io/sclorg/postgresql-16-c9s:c9s
    - name: quay.io/sclorg/redis-7-c9s:c9s
    - name: docker.io/apache/kafka:3.9.1
    - name: docker.io/chrislusf/seaweedfs:3.80
EOF
```

Drop the last two dependency lines, and the PostgreSQL and Redis lines, when you will not run those stand-ins. Drop `koku-ui-onprem:latest` and `nginx-126` when the nodes have AVX2 and you are mirroring a published UI tag instead. Add that published tag as an `additionalImages` entry.

A dry run lists what would be copied without writing the archives:

```bash
oc mirror -c imageset-config-cost-mgmt.yaml file://out --dry-run --v2
```

## 3. Mirror to disk

This runs on the connected bastion. It writes archives under `out/`. Transfer that directory, plus `koku-ui-lab-c9s.tar.gz` if you built it, on media the disconnected registry host can read.

```bash
oc mirror -c imageset-config-cost-mgmt.yaml file://out --v2
```

## 4. Mirror disk to the registry

On a host that can reach the mirror registry, the auth file must contain that registry's credentials. Replace the registry hostname.

```bash
oc mirror -c imageset-config-cost-mgmt.yaml \
  --from file://out \
  docker://registry.example.com:8443 \
  --v2
```

oc-mirror writes cluster resources under `out/working-dir/cluster-resources/`. Those include an ImageDigestMirrorSet, an ImageTagMirrorSet, and a CatalogSource for the redhat index that contains the metrics operator. They do not include a CatalogSource for the service operator. You add that in section 6.

## 5. Point the cluster at the mirror

Apply the generated resources. Do not edit the image fields inside them. Do not apply `signature-configmap.json` when you mirrored operators and did not mirror a platform release.

```bash
oc apply -f out/working-dir/cluster-resources/
oc get imagedigestmirrorset
oc get imagetagmirrorset
oc get catalogsource -n openshift-marketplace
```

The cluster pull secret has to authenticate to the mirror or every pull fails with unauthorized.

```bash
oc get secret/pull-secret -n openshift-config \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq '.auths | keys'
```

The keys must include your mirror host. Merge credentials with the usual disconnected procedure (`oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=...`) when the host is missing. Adding a private CA with `image.config.openshift.io` `additionalTrustedCA` restarts nodes. On a single-node cluster that is a reboot. Plan for it.

## 6. Service operator catalog

The generated CatalogSource covers the redhat index. The service operator still needs its own CatalogSource. Set `spec.image` to the mirrored location of `koku-service-operator-catalog:v0.0.1`. With an ImageTagMirrorSet in place, the upstream name sometimes works. If the catalog pod cannot pull, use the mirror hostname and path that oc-mirror created.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: koku-service-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: Cost Management
  image: registry.example.com:8443/project-koku/koku-service-operator-catalog:v0.0.1
  publisher: Red Hat
  sourceType: grpc
  updateStrategy:
    registryPoll:
      interval: 30m
EOF

oc get pods -n openshift-marketplace -l olm.catalogSource=koku-service-operator-catalog
oc get packagemanifest koku-service-operator
```

Subscribe in `openshift-operators`. That is the only install mode the CSV allows.

```bash
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
```

For the metrics operator, note the CatalogSource name oc-mirror created for the redhat index. The connected guide assumes `redhat-operators`. If your mirrored catalog has another name, use that name as `source` when you create the subscription in `cost-onprem`.

```bash
oc get catalogsource -n openshift-marketplace
```

## 7. Image names in the manifests

With the mirror sets applied, keep the upstream image names in the custom resource and in the dependency manifests. The node rewrites the pull to the mirror. Changing every repository to the mirror hostname is only needed when a registry is missing from the mirror sets, which is common for `docker.io`.

| Situation | What to set |
|---|---|
| UI tag `lab-c9s` pushed to your registry | `ui.app.image` repository and tag of that push, `pullPolicy: IfNotPresent` |
| UI tarball loaded on the node | Upstream name `quay.io/insights-onprem/koku-ui-onprem:lab-c9s`, `IfNotPresent` |
| No mirror set for `docker.io` | Kafka and SeaweedFS image fields set to `registry.example.com:8443/...` |
| You already run PostgreSQL 16, Redis, Kafka, S3, and Keycloak | `deploy: false` and those hosts. Do not deploy the stand-ins. |

## 8. Services that must exist inside the environment

These are not created by mirroring.

| Dependency | What the custom resource expects | Lab stand-in |
|---|---|---|
| PostgreSQL 16 | Databases `costonprem_koku`, `costonprem_rbac`, `costonprem_ros`, `costonprem_kruize`. User `koku` superuser. Extension `pg_stat_statements`. | `quay.io/sclorg/postgresql-16-c9s:c9s` |
| Redis-compatible cache | Authentication on. Secret key `redis-password`. | `quay.io/sclorg/redis-7-c9s:c9s`, service name `valkey` |
| Kafka | Bootstrap `cost-kafka.cost-deps.svc.cluster.local:9092`. Topic `platform.upload.announce`. | `apache/kafka:3.9.1`, KRaft, one broker |
| S3 API | Endpoint service `minio`, port 9000, bucket `koku-bucket`, path-style, region `us-east-1` | SeaweedFS `docker.io/chrislusf/seaweedfs:3.80` |
| OpenID Connect | Realm, UI client `cost-management-ui`, metrics client id equal to a UUID, scope `api.console`, lightweight token claims for `org_id` and `account_number` | Existing Keycloak |

Pods must resolve the identity-provider hostname. On OpenShift, forward only that name. Forwarding the parent zone breaks `*.apps`.

**Lab example.** Keycloak was `https://kc01.dragon.internal`, realm `kubernetes`. Cluster DNS forwarded only that name to `192.168.1.10`. The CA was downloaded from `https://idm01.dragon.internal/ipa/config/ca.crt`, not from the Keycloak TLS handshake. The handshake contained only the leaf.

## 9. Continue with the connected guide

From [cost-management-self-managed.md](./cost-management-self-managed.md), run the sections in this order. Skip section 10's CatalogSource and service subscription if you already applied them above. Skip section 11 if you pushed `lab-c9s` in section 1 of this guide, and set the UI image to the mirror path.

1. Log in and check the CPU.
2. Namespaces and secrets.
3. PostgreSQL, cache, Kafka, and S3, or point the custom resource at services you already have.
4. Keycloak clients, the `costadmin` user, and the identity-provider CA.
5. Cluster DNS for the identity-provider hostname.
6. Service operator, if you did not subscribe it in section 6.
7. UI image load, if you did not push it.
8. `CostManagementServiceConfig`.
9. Metrics operator in `cost-onprem`.
10. Tenant and the metrics service-account grant to `Default admin access`.
11. Metrics custom resource. Apply it only after the tenant and the grant.
12. Sign in, then grant `costadmin` `Default admin access` and restart `cost-onprem-koku-api`.
13. Confirm a kept upload, then create a price list and a cost model.

The same deviations from the knowledgebase sample apply here:

1. The service custom resource lives in `openshift-operators`.
2. Metrics `create_source` is `true`.
3. The metrics client id is a UUID, and the client has default scope `api.console`.
4. `org_id` and `account_number` are on the lightweight access token.
5. The Koku customer is created by hand. The metrics service account and `costadmin` are both added to `Default admin access`. Restart the API after the human grant.
6. `validate_cert` is `false` unless you trust the identity-provider CA cluster-wide.
7. `upload_cycle` below 60 is stored as 60. The first accepted upload can still be discarded. The next cycle, about an hour later, is the one that stays.

## 10. Check the result

```bash
oc get csv -n openshift-operators koku-service-operator.v0.0.1
oc get csv -n cost-onprem -l operators.coreos.com/costmanagement-metrics-operator.cost-onprem=
oc get cmsc cost-onprem -n openshift-operators
oc get costmanagementmetricsconfig -n cost-onprem costmanagementmetricscfg-sample \
  -o jsonpath='source_defined={.status.source.source_defined} upload={.status.upload.last_upload_status}{"\n"}'
```

You want both CSVs at `Succeeded`, the service custom resource at `Ready`, `source_defined=true`, and `last_upload_status=202 Accepted`. Settings in the UI stays locked until `costadmin` is in `Default admin access` and the API has been restarted.
