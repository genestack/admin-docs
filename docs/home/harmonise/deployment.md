# Harmonise deployment

Harmonise lets Curators prepare the raw files of a study for ODM with the help of a large language model (LLM). Each Harmonise run is a Pod that the `applications` service starts through the Kubernetes API, in the release namespace, and steers over a gRPC stream. The run Pod reads the study's files from S3, calls ODM's REST API with a token of the Curator who started it, and calls the LLM provider the Curator chose, with the Curator's own API key.

This page covers what a deployment needs for Harmonise: its settings in the Helm chart, the Kubernetes permissions, and the network access of the run Pods.

!!! note
    Harmonise is always on: there is no flag to enable it. Its settings have defaults in the chart, and only the storage the runs read has to be set for your infrastructure, with the StorageClass of their temporary directory where the cluster's default does not fit (see [Temporary directory](#temporary-directory)).

## Components

| Component | Role |
| --- | --- |
| `applications` | Serves the Harmonise REST API at `/api/v1/harmonisation`, through `nginx`; starts and deletes the run Pods through the Kubernetes API; serves the run streams over gRPC, on port 9090. |
| Run Pods | One per run, named `harmonisation-run-<run id>` and labelled `genestack.com/kind: harmonisation-run`, in the release namespace beside processors-controller's jobs. Image `genestack/harmonisation`. They mount no ServiceAccount token and drop every Linux capability. Each mounts a volume of its own over `/tmp`, its temporary directory, created and deleted with the Pod. |
| ClickHouse, database `harmonisation` | The history of every run, kept for good; the LLM providers and their models; each Curator's LLM credentials; and the record of every run's API token, never the token itself. |

The LLM providers and models Curators choose from are data in ODM, not chart values: users with the Harmonisation Administrator permission manage them in the ODM user interface. Every instance starts with two, Anthropic (`https://api.anthropic.com`) and OpenAI (`https://api.openai.com`).

## Settings: `frontend.harmonisation`

Harmonise is configured by `frontend.harmonisation` in `applications`' `application.yaml`, under `applications.files` in the chart's values:

```yaml
applications:
  files:
    "/var/lib/genestack/properties/application.yaml":
      frontend:
        harmonisation:
          s3:
            bucket: "<bucket>"
            region: "<region>"
            access-key-id: "<access key id>"
            secret-access-key: "<secret access key>"
```

Helm merges these values with the chart's own, so set only the keys you change.

| Key | Default | Description |
| --- | --- | --- |
| `image.repository` | `<applications.image.registry, or global.imageRegistry>/genestack/harmonisation` | Registry and repository of the image run Pods start, without a tag. |
| `image.tag` | empty | The image's tag. Empty: the version of `applications` itself. |
| `kubernetes.namespace` | the release namespace | Namespace of the run Pods. The chart's Role and NetworkPolicies cover only the release namespace: if you change it, extend them to this one. |
| `grpc.port` | `9090` | Port `applications` serves the run streams on. |
| `grpc.target` | `<fullname>-applications:<grpc.port>` | Where run Pods dial the run streams. |
| `grpc.reconnect-window` | `15m` | How long a run resumed after a restart of `applications` waits for its Pod to reconnect. See [Restarts of `applications`](#restarts-of-applications). |
| `clickhouse.url` | `jdbc:clickhouse://<ClickHouse hosts>:8123/harmonisation?createDatabaseIfNotExist=true&clickhouse.jdbc.v1=true` | The run history database. |
| `clickhouse.username`, `clickhouse.password` | `default`, empty | ClickHouse credentials for the run history. |
| `pod.active-deadline-seconds` | `3600` | How long a run Pod may run before Kubernetes stops it. |
| `pod.memory` | `256Mi` | Memory request and limit of the run container. |
| `pod.image-pull-policy` | `IfNotPresent` | Pull policy of the run container. |
| `pod.image-pull-secrets` | the chart's pull secrets, as every Pod of the chart gets | Pull secrets of the run Pods, comma-separated. |
| `pod.temporary-directory.size` | `10Gi` | Size of each run Pod's temporary directory. |
| `pod.temporary-directory.storage-class` | empty | StorageClass of the temporary directory. Empty: the cluster's default. |
| `pod.node-selector` | empty | Node labels a run Pod's node must have: the Pod's `nodeSelector`. Empty, as `pod.tolerations` and `pod.affinity`: any untainted node. See [Node placement](#node-placement). |
| `pod.tolerations` | empty | The run Pods' `tolerations`, for the taints of the nodes they may run on. |
| `pod.affinity` | empty | The run Pods' `affinity`, field names in camelCase as in a Pod's spec. |
| `pod.metadata.labels` | empty | Labels each run Pod and its temporary directory's claim get besides their own. |
| `pod.metadata.annotations` | empty | Annotations each run Pod gets besides its own. |
| `odm.url` | `http://<fullname>-nginx:<nginx.service.http.port>` | ODM's API gateway, as run Pods call it. |
| `s3.bucket` | empty | The bucket a run lists when the Curator names none. |
| `s3.endpoint` | empty | The S3 endpoint, for S3-compatible storage such as MinIO. Empty for AWS S3. |
| `s3.region` | empty | The bucket's region. |
| `s3.access-key-id`, `s3.secret-access-key` | empty | The access key run Pods read S3 with. |

`<fullname>` is the chart's full name: `fullnameOverride` if set, otherwise the release name if it contains `odm`, otherwise `<release>-odm`.

### Image

The run Pods start `genestack/harmonisation`, from the same registry as the other ODM images, with the chart's pull secrets. Leave `image.tag` empty: `applications` then starts the image of its own version, which every ODM release publishes. If you mirror ODM's images into your own registry, mirror this one too.

A run that cannot pull its image stays `PENDING`, and the Curator sees the reason, such as `ErrImagePull`.

### gRPC port 9090

`applications` serves the run streams on port 9090 (`grpc.port`), which the chart adds to its container and to its Service, as the port `grpc`. Only the run Pods dial it.

The stream is neither authenticated nor encrypted: a run Pod identifies itself only by its run id, and `applications` trusts it. Run ids are sequential, and appear in the run Pods' names. So any Pod that reaches the port and names an open run's id is answered as that run. For a run started since `applications` last restarted, it gets the run's token, which acts in ODM as the Curator who started the run, the S3 access key, and that Curator's LLM key. For a run resumed after a restart, it gets the S3 access key only: `applications` held the token and the LLM key in memory only (see [Restarts of `applications`](#restarts-of-applications)). Once a run is over, its stream carries none of the three. Limit which Pods reach the port:

- **Keep `applications.service.type` at `ClusterIP`.** With `NodePort` or `LoadBalancer`, the port is exposed outside the cluster too.
- **Set `networkPolicy.enabled: true`** where Harmonise is used. It is `false` by default, and then any Pod in the cluster can reach the port. With it, the chart's ingress policy admits the Pods of the release namespace, processors-controller's transformation jobs and the run Pods included, and those of the `monitoring` and `clickhouse-operator` namespaces.

### S3 for run Pods

Runs read study files from S3, or from S3-compatible storage. `applications` sends each run Pod the endpoint, region and access key over its stream, never in the Pod's spec, and the Curator names the bucket and path to read, `s3.bucket` by default.

Run Pods mount no ServiceAccount token, so IAM roles for service accounts, or workload identity, do not apply to them: set `s3.access-key-id` and `s3.secret-access-key`. This key is the deployment's, not a run's: every run gets it, and so does anything that reaches the run stream (see [gRPC port 9090](#grpc-port-9090)), and nothing revokes it when a run ends. Curators can name any bucket, so the key's own policy is the only limit on what runs read: make it a read-only key limited to the bucket the runs read.

With either half of the key left empty, the run Pod falls back to the AWS SDK's default credential chain, which on a cloud node may pick up the node's own role from the instance metadata service.

These settings are in `application.yaml`, which the chart renders into a ConfigMap, the secret access key included. Restrict who may read ConfigMaps in the release namespace accordingly.

When a run imports files into ODM, ODM reads them itself, from the same bucket, so the bucket must also be one ODM can import from.

### ClickHouse database `harmonisation`

`applications` keeps Harmonise's data in the ClickHouse database `harmonisation` (`clickhouse.url`). It creates the database on the cluster and migrates it at startup with its own changelog. The database holds:

- the history of every run: its status changes, every message between `applications` and the run Pod, and every request of the Curator, with who and when. Runs are kept for good, with no TTL.
- the LLM providers and their models, in `LlmProvider`.
- each Curator's LLM credentials, in `UserLlmCredentials`, with the API key in plaintext (see [Security notes](#security-notes)).
- the record of every run's API token, never the token itself, in `RunToken`. Deleting a token adds its id to `RunTokenArchive` and leaves its record.

A change to a provider or to credentials writes a new version of its row. A delete archives the entry in `LlmProviderArchive` or `UserLlmCredentialsArchive`.

Include the database in your ClickHouse backups. The backups then hold every Curator's LLM key.

`applications` does not start without `clickhouse.url`, which the chart sets.

### Active deadline

A run stays open until its Curator finishes it. `pod.active-deadline-seconds` (one hour by default) bounds it: Kubernetes stops a run Pod still running after that long, and the run is `FAILED`. It also ends a step that never does. Raise it if your Curators' runs need longer.

### Temporary directory

Each run Pod gets a temporary directory for its intermediate results: a volume of its own, mounted over `/tmp`, which the pipeline finds in its `TEMPORARY_DIRECTORY` environment variable. It is a [generic ephemeral volume](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/#generic-ephemeral-volumes): Kubernetes creates its PersistentVolumeClaim, `harmonisation-run-<run id>-tmp`, with the run Pod's labels, those of `pod.metadata.labels` included, when the Pod is created, and deletes it with the Pod. So a run's files go with its Pod, and the claim needs no permission beyond the shared Role: Kubernetes creates it, not `applications`.

The volume holds `pod.temporary-directory.size`, 10Gi by default. A step that fills it fails, with an error that starts `No space left` and says how much the directory holds, and the run stays open for the Curator's next step. Raise the size if your Curators' runs need more.

Set `pod.temporary-directory.storage-class` to a StorageClass, or check that the cluster's default one:

- **provisions volumes dynamically.** Otherwise the claim is never bound, and the run stays `PENDING` with `Unschedulable`. Prefer `volumeBindingMode: WaitForFirstConsumer`: with `Immediate`, on a cluster across zones, a volume can be created in a zone where the Pod cannot run.
- **deletes a volume with its claim**, with `reclaimPolicy: Delete`. With `Retain`, every run leaves a volume behind. On Amazon EKS, a gp3 class with `reclaimPolicy: Delete` fits.
- **enforces the size**, as block storage such as Amazon EBS does. With a provisioner that does not, such as a local-path or hostPath one, a run fills the node's disk instead of failing its step.
- **lets the Pod write to the new volume.** Run Pods run as user and group 999, with `fsGroup: 999`, and the volume must give that group write access, as Amazon EBS does. With a class that leaves the volume owned by root, such as one whose CSI driver ignores `fsGroup`, runs still start, but a write to `/tmp` fails with `Permission denied`, and the run fails with it. Files that libraries create through Python's `tempfile` go to the container's own filesystem instead, outside the size.

Each run Pod holds one volume, which counts towards the node's limit of attached volumes and any storage quota of the namespace; a failed run's claim stays, and counts towards the quota, until the next cleanup.

### Node placement

Kubernetes places run Pods on any untainted node by default. `pod.node-selector`, `pod.tolerations` and `pod.affinity` set where, as processors-controller's `transformations.pod` does for transformation jobs. `applications` copies them into every run Pod's spec, as its `nodeSelector`, `tolerations` and `affinity`. `pod.metadata.labels` and `pod.metadata.annotations` add labels and annotations to every run Pod, and the labels to its temporary directory's claim too.

To run them on a node group of their own, labelled `harmonisation.node: "true"` and tainted `harmonisation.node=true:NoSchedule`, and to leave them out of Velero's backups:

```yaml
applications:
  files:
    "/var/lib/genestack/properties/application.yaml":
      frontend:
        harmonisation:
          pod:
            node-selector:
              harmonisation.node: "true"
            tolerations:
              - key: harmonisation.node
                operator: Equal
                value: "true"
                effect: NoSchedule
            metadata:
              labels:
                "[velero.io/exclude-from-backup]": "true"
```

To share the transformation jobs' nodes instead, set `pod.node-selector` to their `transformations.pod.node_selector`, such as `worker.node: "true"`.

The values are Kubernetes' own YAML, with one exception for map keys. A key holding a character other than a letter, a digit, `-` or `.` goes in brackets, quoted: `"[karpenter.sh/nodepool]": "harmonisation"`. Spring, which reads `application.yaml`, drops such characters from a bare key, with no error. So `karpenter.sh/nodepool` becomes `karpenter.shnodepool`, which no node has. Keys with dots only, such as `worker.node`, need no brackets. The rule covers map keys only: the `key` of a toleration or of a `matchExpressions` entry is a value, written as is.

The run's own labels, `genestack.com/kind`, `genestack.com/harmonisation-run-id` and `app.kubernetes.io/managed-by`, and its annotation `genestack.com/harmonisation-started-by` take precedence over configured ones of the same name. `applications` finds its run Pods, and who started each, by them.

A node group in fewer zones than the cluster needs a temporary directory StorageClass with `volumeBindingMode: WaitForFirstConsumer` (see [Temporary directory](#temporary-directory)). With `Immediate`, the volume can be created in a zone with no node of the group, and the run stays `PENDING`.

### Cleanup

`applications` deletes a run's Pod once the run has shut down, and deletes the run's API token. Every 15 minutes, it also deletes the Pods and tokens of runs that are over, such as failed ones.

### Restarts of `applications`

An open run survives a restart of `applications`, such as a Helm upgrade. The restart drops every run stream, and each run Pod retries its stream for 15 minutes. At startup, `applications` rebuilds every open run whose Pod had reported ready, from its history in ClickHouse. The run keeps its Pod and its token, and goes on once its Pod opens a new stream.

- **A run whose Pod had not reported ready** becomes `FAILED`, and its Pod is deleted.
- **A resumed run whose Pod has not reconnected** within `grpc.reconnect-window` of the restart, 15 minutes by default, becomes `FAILED`, and its Pod is deleted.

`applications` held each run's token and its Curator's LLM credentials in memory only. After a restart, the stream no longer carries them, and the run Pod keeps those it had.

The 15 minutes cover a Helm upgrade, in which the chart allows `applications` up to 10 minutes to start. Keep `grpc.reconnect-window` at 15 minutes or more: a shorter one fails runs whose Pods are still retrying.

## Kubernetes permissions: the shared Role

`applications` and processors-controller run on the release's shared ServiceAccount, and manage their Pods with one Role, `<fullname>-processors-controller`, and its RoleBinding:

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/exec", "configmaps", "persistentvolumeclaims"]
    verbs: ["*"]
```

The chart renders them whenever `processorsController.rbac.enabled` is `true`, the default, even with processors-controller itself off (`processorsController.enabled: false`). The Role covers every Pod in the release namespace; `applications` touches only the Pods labelled as runs.

If you create RBAC yourself (`processorsController.rbac.enabled: false`), grant the release's ServiceAccount the same rules in the run Pods' namespace, or `applications` cannot start runs. Harmonise needs no permissions beyond those processors-controller already has.

`applications` uses its in-cluster ServiceAccount for the Kubernetes API, so it needs egress to the API server; the chart's egress NetworkPolicy leaves `applications` out for that reason.

## Network access of run Pods

Run Pods connect to:

| Destination | Why |
| --- | --- |
| kube-dns, in `kube-system`, port 53 (UDP and TCP) | Name resolution for every destination below. |
| `applications`, port 9090 (`grpc.target`) | The run stream. |
| `nginx` (`odm.url`) | ODM's REST API, with the Curator's token. |
| The S3 endpoint (`s3.endpoint`, or AWS S3's regional endpoint) | The study files. |
| Each LLM provider's base URL, over HTTPS | The LLM calls, with the Curator's API key. |

The chart's NetworkPolicies select Pods by `app.kubernetes.io/instance`, which run Pods do not carry unless `pod.metadata.labels` adds it, so the chart does not restrict their egress. If your cluster does (a default-deny NetworkPolicy, an egress firewall), allow traffic from Pods labelled `genestack.com/kind: harmonisation-run` to these destinations.

### Egress to the LLM providers

Run Pods call the LLM provider of the run's Curator directly, from inside the cluster. By default that is `api.anthropic.com` or `api.openai.com`, on port 443. A provider a Harmonisation Administrator adds in ODM needs the same egress to its base URL: agree with them which providers to offer.

These requests leave your network. Offer only providers your organisation accepts for the data your Curators work with.

## Security notes

- **LLM keys are stored in plaintext** in ClickHouse, in `harmonisation.UserLlmCredentials`. Anyone who can read the database or its backups can read every Curator's key. Deleting or replacing credentials writes a new version of the row, without the old key. Earlier versions keep that key until ClickHouse merges the rows, in the background, at a time it chooses. ODM never returns a key through its API or user interface.
- **The run stream is not authenticated or encrypted.** A Pod that reaches port 9090 and names the id of a run started since `applications` last restarted gets the run's token, which acts as its Curator in ODM, the S3 access key, and the Curator's LLM key. For a run resumed after a restart, it gets the S3 access key. With `networkPolicy.enabled: false`, the chart's default, any Pod in the cluster can reach the port. See [gRPC port 9090](#grpc-port-9090).
- **The S3 access key is the deployment's**: every run gets the same one. See [S3 for run Pods](#s3-for-run-pods).

## Troubleshooting

List the run Pods, and read a run's log, which the Pod writes to its standard error only:

```shell
kubectl -n <namespace> get pods -l genestack.com/kind=harmonisation-run
kubectl -n <namespace> logs harmonisation-run-<run id>
```

`kubectl logs` works only while the Pod exists: a finished run's Pod is deleted once it has shut down, and a failed run's at the next cleanup, within 15 minutes. After that, the log is only in your cluster's log collection, if it has one.

- **A run stays `PENDING` with `ErrImagePull`:** the image, its tag or the pull secrets (see [Image](#image)).
- **A run stays `PENDING` with `Unschedulable`, or `SchedulerError`:** Kubernetes cannot schedule its Pod. Most often, its temporary directory's claim cannot be bound: no default StorageClass and no `pod.temporary-directory.storage-class`, a class that cannot provision, or the namespace's storage quota (see [Temporary directory](#temporary-directory)). Otherwise, no node matches `pod.node-selector`, `pod.tolerations` and `pod.affinity`, or a node group that scales from zero is still starting one (see [Node placement](#node-placement)). `kubectl -n <namespace> describe pvc harmonisation-run-<run id>-tmp` and `kubectl -n <namespace> describe pod harmonisation-run-<run id>` say why. Every run is `Unschedulable` for a moment as it starts, while Kubernetes creates the claim.
- **A run fails with `Permission denied` on `/tmp` in its Pod's log:** check that the temporary directory's StorageClass gives the Pod's group write access to a new volume (see [Temporary directory](#temporary-directory)).
- **A run stays `PENDING` with no reason while its Pod is `Running`, then becomes `FAILED`:** the run Pod cannot reach the run stream. Check `grpc.target`, and any policy between the run Pods and port 9090. The Pod keeps trying for 15 minutes from its first failed attempt, then exits with code 2.
- **A run is `FAILED` after a restart of `applications`:** its Pod had not reported ready, or did not reconnect within `grpc.reconnect-window` (see [Restarts of `applications`](#restarts-of-applications)).
- **The Harmonise API answers 503:** `applications` cannot reach the Kubernetes API or ClickHouse; check its log.
- **A run's LLM check fails with `LLM_UNREACHABLE`:** the run Pod cannot reach the provider (see [Egress to the LLM providers](#egress-to-the-llm-providers)).
- **A run cannot list files:** the S3 settings, or the run Pod's egress to the S3 endpoint (see [S3 for run Pods](#s3-for-run-pods)).
- **A step fails with `No space left`:** the run's temporary directory is full. Raise `pod.temporary-directory.size` (see [Temporary directory](#temporary-directory)).
