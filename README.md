# OpenShift cost management self-managed beta

This repository is a walkthrough for installing Red Hat Lightspeed cost management self-managed on OpenShift 4.22. It records a working install so you can repeat it on your own cluster. Substitute your API, apps domain, storage class, and identity provider. Notes marked **Lab example** in the guides are one cluster that proved the steps. They are not values to copy.

The product article is [Knowledgebase 7148128](https://access.redhat.com/articles/7148128). The operator source is [project-koku/koku-server-operator](https://github.com/project-koku/koku-server-operator). That repository is not the install path. Install from the catalog image and from `redhat-operators`, as the guides do.

This beta reports OpenShift usage cost. It does not ingest AWS, Azure, or GCP bills. Resource optimization stays off.

## Which guide

| Your cluster | Start here |
|---|---|
| Can pull images from the internet | [cost-management-self-managed.md](./cost-management-self-managed.md) |
| Cannot pull images from the internet | [cost-management-airgapped.md](./cost-management-airgapped.md), then the connected guide |

A disconnected install is the same procedure with a mirror in front of it. Mirror the images, point the cluster at that registry, then continue at namespaces in the connected guide.

## What you provide

The service operator does not deploy its database, cache, broker, object store, or identity provider. Those have to exist before the custom resource:

- PostgreSQL 16
- a Redis-compatible cache
- Kafka
- an S3 API
- an OpenID Connect provider (the guides use Keycloak)

You also install two operators, in two different namespaces:

| Operator | Catalog | Namespace |
|---|---|---|
| Cost Management Service Operator `0.0.1` (`koku-service-operator`, channel `beta`) | `quay.io/project-koku/koku-service-operator-catalog:v0.0.1`. This catalog is not in `redhat-operators`. | `openshift-operators` only |
| Cost Management Metrics Operator (`costmanagement-metrics-operator`, channel `stable`, verified at `4.4.2`) | `redhat-operators` | `cost-onprem` only |

## Installation flow

```text
Choose the guide
        |
        +-- disconnected: mirror images, then apply the mirror sets
        |
        v
Log in (:6443) and check the CPU
        |
        v
Namespaces and secrets
        |
        v
PostgreSQL, cache, Kafka, S3
        |
        v
Keycloak: realm, costadmin, UI client, metrics client
        |
        v
Cluster DNS for the identity provider only
        |
        v
Service operator, then the UI image
        |
        v
CostManagementServiceConfig  -->  wait until Ready
        |
        v
Metrics operator
        |
        v
Create the tenant and grant the metrics service account
Default admin access
        |
        v
Metrics custom resource  -->  first upload may be dropped;
                               the next cycle (~60 min) is kept
        |
        v
Sign in as costadmin, grant that user Default admin access,
restart the API
        |
        v
Price list and cost model  -->  dollar totals stay at 0 until this exists
```

Three points in that sequence are easy to do too early:

1. Apply the metrics custom resource only after the tenant and the metrics service-account grant. The operator's first call is a GET, and the tenant is created on a non-GET, so an early upload is refused.
2. Sign in as `costadmin`, then add that user to **Default admin access** and restart `cost-onprem-koku-api`. Until then, Settings shows "You do not have access to Cost management".
3. A price list and a cost model come last. Usage can be present while every dollar total is still zero.

On a CPU without AVX2, the published UI image will not start. The connected guide rebuilds it onto UBI 9 before the custom resource is applied. Check the CPU in the first step and follow that branch only when `avx2` is missing.

## After it is up

Both operator CSVs are `Succeeded`. `CostManagementServiceConfig` is `Ready`. The metrics status shows `source_defined=true` and `last_upload_status=202 Accepted`. The UI is `https://cost-onprem-ui-openshift-operators.<apps-domain>`.

Keep passwords, client secrets, and private keys out of git. The guides store them in a file outside the repository.
