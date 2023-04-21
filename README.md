# GKE Pgbouncer

PgBouncer is a lightweight connection pooler for PostgreSQL. This chart is forked from https://github.com/wallarm/pgbouncer-chart, and tailored to be used on Google Kubernetes Engine by using Cloud Proxy.

## Usage

### Clone this project

Clone this repo to a directory, for this example `/app/pgbouncer-template`

```bash
~$ git clone git@github.com:8grams/pgbouncer-chart-gke.git /app/pgbouncer-template
```

### values.yml

Create a file named `values.yml` which contains the following configuration. Please adjust to your appropriate values.
This example assumes we have:
1. A PostreSQL CloudSQL Instance named: `my-db-instance`, in region `asia-southeast2`
2. A Database, inside `my-db-instance`, called `my-db`
3. DB User called `admin` with password `xxx`
4. Kubernetes Service Account associated with a CloudSQL's service account named `kubesql-ksa`. See `Annotate GKE Service Account` section below for more details.

```
config:
  adminUser: admin
  adminPassword: xxx
  databases: 
    # your database name
    my-db:
      host: 127.0.0.1
      port: 5433
  pgbouncer: 
    auth_type: md5
    pool_mode: transaction
    max_client_conn: 1024
    default_pool_size: 15
    max_db_connections: 15
    listen_port: 5432
    ignore_startup_parameters: extra_float_digits

replicaCount: 1
updateStrategy:
  type: RollingUpdate

minReadySeconds: 0
revisionHistoryLimit: 3
imagePullSecrets: []

image:
  registry: ""
  repository: pgbouncer/pgbouncer
  tag: 1.15.0
  pullPolicy: Always

service:
  type: ClusterIP
  port: 5432

pgbouncerExporter:
  enabled: false

# Kubernetes Service Account associated with a CloudSQL's service account
serviceAccount:
  name: kubesql-ksa
  annotations: {}
  create: false

extraContainers:
- name: cloud-sql-proxy
  image: gcr.io/cloudsql-docker/gce-proxy:1.17
  command:
    - "/cloud_sql_proxy"
    - "-ip_address_types=PRIVATE"

    # your GCP project ID, region, and DB Instance's name
    - "-instances=my-gcp-project:asia-southeast2:my-db-instance=tcp:5433""
  securityContext:
    runAsNonRoot: true
```

### Install using Helm

```
~$ helm install pgbouncer /app/pgbouncer-template -f values.yaml --namespace <your-namespace>
```


### Override PgBouncer Setting

The setting about pgbouncer lies in `config.pgbouncer`. This reflects to `pgbouncer.ini` files in keys-values manner. For details, go to https://www.pgbouncer.org/config.html.

```yaml
config:
  pgbouncer:
    auth_type: md5
    pool_mode: transaction
    max_client_conn: 1024
    default_pool_size: 15
    max_db_connections: 15
    listen_port: 5432
```

### Annotate GKE Service Account

GKE Service Account must be annotated with Cloud SQL Service Account in order to work with PgBouncer.

Create namespace if does not exist

```
~$ kubectl create namespace myapp
```

Create Kubernetes Service Account (e.g `kubesql-ksa`) on that namespace

```
~$ kubectl create serviceaccount --namespace myapp kubesql-ksa
```

Let's say you have a GCP Project `my-gcp-project`, and a Cloud SQL Service Account `cloudsql-sa`.

```
~$ gcloud iam service-accounts add-iam-policy-binding \
 --role roles/iam.workloadIdentityUser \
 --member "serviceAccount:my-gcp-project.svc.id.goog[myapp/kubesql-ksa]" \
 cloudsql-sa@my-gcp-project.iam.gserviceaccount.com
```


Annotate Kubernetes Service Account with Cloud Service Account

```
~$ kubectl annotate serviceaccount \
--namespace myapp kubesql-ksa \
iam.gke.io/gcp-service-account=cloudsql-sa@my-gcp-project.iam.gserviceaccount.com
```


## Chart Configuration

The following table lists the configurable parameters of the Prometheus chart and their default values.

Parameter | Description | Default
--------- | ----------- | -------
`replicaCount`      | Desired number of pgbouncer pods | `1`
`updateStrategy`    | Deploy strategy of the Deployment | `{}`
`minReadySeconds`   | Interval between discrete pods transitions | `0`
`revisionHistoryLimit` | Rollback limit | `10`
`imagePullSecrets`  | Container image pull secrets | `[]`
`image.registry`    | Pgbouncer container image registry | `""`
`image.repository`  | Pgbouncer container image repository | `pgbouncer/pgbouncer`
`image.tag`         | Pgbouncer container image tag | `1.15.0`
`image.pullPolicy`  | Pgbouncer container image pull policy | `IfNotPresent`
`service.type`      | Type of pgbouncer service to create | `ClusterIP`
`service.port`      | Pgbouncer service port | `5432`
`podLabels`         | Labels to add to the pod metadata | `{}`
`podAnnotations`    | Annotations to add to the pod metadata | `{}`
`extraEnvs`         | Additional environment variables to set | `[]`
`resources`         | Pgbouncer pod resource requests & limits | `{}`
`nodeSelector`      | Node labels for pgbouncer pod assignment | `{}`
`lifecycle`         | Lifecycle hooks | `{}`
`tolerations`       | Node taints to tolerate (requires Kubernetes >=1.6) | `[]`
`affinity`          | Pod affinity | `{}`
`priorityClassName` | Priority of pods | `""`
`runtimeClassName`  | Runtime class for pods | `""`
`config.adminUser`  | Set pgbouncer `admin_user` option. `admin_user` - database user that are allowed to connect and run all commands on console. | `admin`
`config.adminPassword` | Set password for `admin_user` | `""`, required
`config.authUser`   | Set pgbouncer `auth_user` option. If `auth_user` is set, any user not specified in `auth_file` will be queried through the `auth_query` query from `pg_shadow` in the database using `auth_user` | `""`
`config.authPassword` | Set password for `auth_user` | `""`
`config.databases`  | Dict of database connections string described in section `[databases]` in pgbouncer.ini file | `{}`
`config.pgbouncer`  | Dict of pgbouncer options described in section `[pgbouncer]` in pgbouncer.ini file | 
`config.userlist`   | Dict of users for `userlist.txt` file | `{}`
`extraContainers`   | Additional containers to be added to the pods | `[]`
`extraInitContainers` | Containers, which are run before the app containers are started | `[]`
`extraVolumeMounts` | Additional volumeMounts to the main container | `[]`
`extraVolumes`      | Additional volumes to the pods | `[]`
`pgbouncerExporter.enabled` | Enables pgbouncer exporter in pod | `false`
`pgbouncerExporter.port` | Pgbouncer exporter port | `9127`
`pgbouncerExporter.image.registry` | Pgbouncer exporter image registry | `""`
`pgbouncerExporter.image.repository` | Pgbouncer exporter image repository | `prometheuscommunity/pgbouncer-exporter`
`pgbouncerExporter.image.tag` | Pgbouncer exporter image tag | `2.0.1`
`pgbouncerExporter.image.pullPolicy` | Pgbouncer exporter image pull policy | `IfNotPresent`
`pgbouncerExporter.resources` | Pgbouncer exporter resources | `{"limits":{"cpu":"250m","memory":"150Mi"},"requests":{"cpu":"30m","memory":"40Mi"}}`
`serviceAccount.name` | Kubernetes ServiceAccount for service. Creates new if empty | `""`
`serviceAccount.annotations` | Annotations to set for ServiceAccount | `{}`
