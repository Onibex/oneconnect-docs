# Deployment & Installation

Failures that happen while installing or upgrading the Helm chart, or that prevent pods from reaching `Running`.

Applies to the SmartGateway Helm chart on EKS, AKS, and GKE. For SAP BTP / Kyma, see the [BTP deployment troubleshooting](../Smart_Gateway/BTP/SmartGateway_BTP_Kyma_Deployment.md#troubleshooting) — Cloud Connector and Istio issues are covered there. For the installation steps themselves, see the [AWS](../Smart_Gateway/AWS/SmartGateway_Helm_Chart_AWS.md) and [Azure](../Smart_Gateway/Azure/SmartGateway_Helm_Chart_Azure.md) guides.

---

## 0. First, Rule Out "It Is Just Slow"

Before diagnosing anything, know the chart's normal timings:

| Behavior | Normal value | Implication |
|---|---|---|
| Readiness probe initial delay | **300 seconds** on most services | A pod showing `0/1` for the first ~5 minutes is expected |
| MariaDB wait init container (`wait-for-mysql`) | Up to **300 seconds** per attempt | `auth`, `builder`, `cwcback`, `logs`, and `metrics` legitimately sit in `Init:0/1` for minutes |
| MariaDB readiness probe | **200 seconds** initial delay | MariaDB is the long pole on a cold install |
| `frontend-apigateway-sync` post-install hook | Waits up to **600 s** for the API Gateway load balancer address, then up to **10 min** for the frontend rollout | Helm waits for this hook before reporting success |
| Recommended install timeout | `--timeout 15m` | Gives the post-install hook the time it needs (see [§9](#9-helm-reports-failed-post-install-timed-out-waiting-for-the-condition)) |

```bash
# Watch rather than re-running helm
kubectl get pods -n oneconnect -w
```

A pod is only a problem if it is `CrashLoopBackOff`, `ImagePullBackOff`, `Pending` beyond a few minutes, or still `0/1` well past its initial delay.

---

## 1. `helm install` Fails Immediately

### Missing required values

Every install uses the same set of flags. The Docker Hub token is supplied at install time, and the install guides expose the UI publicly with `networking.loadBalancer.type=external`:

```bash
helm install oneconnect . \
  --namespace oneconnect --create-namespace \
  --values values-aws.yaml \
  --values secrets.local.yaml \
  --set dockerHub.token=dckr_pat_xxxxxxxxxxxx \
  --set networking.loadBalancer.type=external \
  --timeout 15m
```

> 📌 Always pass `dockerHub.token`; if it is missing, images fail to pull and pods show `ImagePullBackOff`. See [§2](#2-imagepullbackoff--errimagepull).

> 💡 Use `values-aws.yaml`, `values-azure.yaml`, or `values-gcp.yaml` to match your cluster. The cloud file sets `cloudProvider`, the storage classes ([§4](#4-persistentvolumeclaim-problems)), and the load balancer annotations ([§7](#7-no-external-ip--cannot-reach-the-ui)).

### Missing required secrets

**Symptom** — `helm install` (or `helm template` / `helm upgrade`) stops before creating anything with an error such as:

```
Error: INSTALLATION FAILED: execution error at (oneconnect/templates/secrets.yaml:...): mysql.rootPassword is required (or set mysql.existingSecret)
```

**Cause** — the chart ships no credentials. Every install and upgrade must receive them through `secrets.local.yaml` (see the [AWS](../Smart_Gateway/AWS/SmartGateway_Helm_Chart_AWS.md) or [Azure](../Smart_Gateway/Azure/SmartGateway_Helm_Chart_Azure.md) guide) or through existing Kubernetes Secrets.

| Message | Value to provide |
|---|---|
| `mysql.rootPassword is required (or set mysql.existingSecret)` | `mysql.rootPassword` |
| `mysql.password is required (or set mysql.existingSecret)` | `mysql.password` |
| `appSecrets.jwtSecret is required (or set appSecrets.existingSecret)` | `appSecrets.jwtSecret` |
| `appSecrets.softToken is required (or set appSecrets.existingSecret)` | `appSecrets.softToken` |
| `akhq.security.jwtSecret is required` (kafka-platform) | `akhq.security.jwtSecret` in `kafka-platform/secrets.local.yaml` |
| `akhq.security.users[].password (SHA-256) is required` (kafka-platform) | The SHA-256 of the AKHQ admin password (`echo -n 'pwd' \| sha256sum`) |

**Fix** — add `--values secrets.local.yaml` (or `--values kafka-platform/secrets.local.yaml` for `kafka-platform`) to the command, or set `mysql.existingSecret` / `appSecrets.existingSecret` to Secrets you created.

### Render the templates without installing

The fastest way to see what Helm is actually producing:

```bash
helm template oneconnect . --values values-aws.yaml --values secrets.local.yaml \
  --set dockerHub.token=dummy > /tmp/rendered.yaml

helm lint . --values values-aws.yaml --values secrets.local.yaml --set dockerHub.token=dummy
```

### A previous release is stuck

```bash
helm list -n oneconnect --all
```

If the release shows `pending-install`, `pending-upgrade`, or `failed`, it must be cleared before retrying:

```bash
helm rollback oneconnect -n oneconnect          # to the last good revision
# or, if there is no good revision:
helm uninstall oneconnect -n oneconnect
```

> 💡 A release marked `failed` because the post-install hook ran out of time usually has healthy pods. Check [§9](#9-helm-reports-failed-post-install-timed-out-waiting-for-the-condition) before uninstalling.

### Deployment order was not respected

The stack is installed in three phases, each building on the previous one:

| Phase | Component | Gate before proceeding |
|---|---|---|
| 1 | Strimzi operator | `strimzi-cluster-operator` pod shows `1/1 Running` |
| 2 | `kafka-platform` chart | Kafka, Schema Registry, and AKHQ pods all `1/1 Running` |
| 3 | `oneconnect` chart | — |

```bash
kubectl get pods -n kafka | grep strimzi-cluster-operator
kubectl get crd kafkas.kafka.strimzi.io
kubectl get pods -n kafka
```

What you see when a phase is skipped:

| Message | Meaning | Fix |
|---|---|---|
| `no matches for kind "Kafka" in version "kafka.strimzi.io/v1" -- ensure CRDs are installed first` (on `kafka-platform`) | The Strimzi CRDs are not in the cluster yet | Install the Strimzi operator (phase 1) first, then retry phase 2 |
| `namespaces "kafka" not found` (on `oneconnect`) | The `oneconnect` chart also writes the pull secret and the Kafka Connect RBAC into the `kafka` namespace (`kafkaNamespace`), which is created in phase 1 | Complete phases 1 and 2, then retry |

> 📌 Wait for the operator to reach `1/1 Running` before starting phase 2. The Kafka custom resources are reconciled once the operator is ready, so starting early surfaces later as "Kafka pods not starting".

---

## 2. `ImagePullBackOff` / `ErrImagePull`

**Symptom** — Onibex pods (for example `auth`, `builder`, `mysql-0`, `datasynchub`, or the Producer/Consumer pods) show `ErrImagePull` or `ImagePullBackOff`, typically a few minutes after a successful `helm install` or `helm upgrade`.

**Cause** — Usually the Docker Hub token: it was not supplied, has expired, or was not passed again on `helm upgrade`.

### Confirm it

```bash
kubectl describe pod -n oneconnect <pod-name> | tail -30
```

Look for `Failed to pull image ... unauthorized` or `pull access denied`.

### How the secret works

The chart builds a `kubernetes.io/dockerconfigjson` secret named **`docker-hub-credentials`** from these values:

| Value | Default | Notes |
|---|---|---|
| `dockerHub.registry` | `https://index.docker.io/v1/` | Rarely changed |
| `dockerHub.username` | Provided by Onibex together with your token | — |
| `dockerHub.token` | *(empty)* | **You must supply this** on every install and upgrade |
| `dockerHub.createSecret` | `true` | Set to `false` only if you manage the secret yourself |

When `createSecret` is enabled the same secret is created in **three namespaces**: `oneconnect` (platform services, Producer, Consumer), `datasynchub` (DataSyncHub), and `kafka` (the Kafka Connect deployments the Builder creates for each workspace).

### Checks

```bash
# Does the secret exist in all three namespaces?
kubectl get secret docker-hub-credentials -n oneconnect
kubectl get secret docker-hub-credentials -n datasynchub
kubectl get secret docker-hub-credentials -n kafka

# Decode it and confirm the username and token are what you expect
kubectl get secret docker-hub-credentials -n oneconnect \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
```

An empty `"password":""` in the decoded output means the token was not supplied.

### Fixes

| Cause | Fix |
|---|---|
| Token not supplied | Re-run `helm upgrade` with `--set dockerHub.token=...` (and the other flags from your install, see [§8](#8-upgrades)) |
| Token expired or was revoked | Generate a new Docker Hub PAT, then `helm upgrade` with the new value |
| Token pasted with quotes or whitespace | Re-set it cleanly; shell quoting frequently corrupts `dckr_pat_` strings |
| `createSecret: false` but no secret provided | Either set it to `true`, or create `docker-hub-credentials` manually in all three namespaces |
| Secret exists but pods still fail | Delete the pods so they retry: `kubectl delete pod -n oneconnect <pod>` |

After updating the token, the secret changes but running pods do not re-pull. Restart them:

```bash
kubectl rollout restart deployment -n oneconnect
kubectl rollout restart statefulset mysql -n oneconnect
kubectl rollout restart deployment -n datasynchub
```

### `toomanyrequests: You have reached your pull rate limit`

A few supporting components use public images: the database setup Job, `otel-collector`, the `frontend-apigateway-sync` hook, and, in the `kafka` namespace, Schema Registry and AKHQ. If many nodes share one public egress IP, Docker Hub's anonymous pull limit can apply to these.

**Fix:** retry after the limit window, or point these images at a registry mirror your cluster can reach. For the hook, set `networking.loadBalancer.autoConfigureFrontend.kubectlImage` to a reachable `kubectl` image.

### When only the Producer/Consumer pods are affected

Base chart images are pulled once and cached (`imagePullPolicy: IfNotPresent`). Producer and Consumer workloads are created later, when a SAP Link is created, so they are the first to use a token that has since expired.

```bash
kubectl get pods -n oneconnect | grep -Ei 'producer|consumer'
```

If these are the only pods in `ImagePullBackOff`, refresh the token as above.

---

## 3. Pods Stuck in `Pending`

```bash
kubectl describe pod -n oneconnect <pod-name> | grep -A10 Events
```

| Message | Cause | Fix |
|---|---|---|
| `Insufficient cpu` / `Insufficient memory` | Node pool too small | Scale the node group, or lower `services.<service>.resources.requests` / `mysql.resources` in values |
| `pod has unbound immediate PersistentVolumeClaims` | Storage class missing or no provisioner | See [§4](#4-persistentvolumeclaim-problems) |
| `had untolerated taint` | Nodes are tainted | Add matching entries under `scheduling.tolerations` in values (see the scope note below) |
| `didn't match node affinity` | `scheduling.nodeAffinity` rules exclude all nodes | Correct the rules; they apply only with `scheduling.nodeAffinity.enabled: true` |

As a sizing reference, the `oneconnect` chart requests roughly **4.2 GiB of memory and 1 vCPU** with the defaults, and the Kafka broker requests 2 GiB on its own. Each workspace created later adds its own Producer, Consumer, and Kafka Connect pods.

Check capacity:

```bash
kubectl get nodes
kubectl describe nodes | grep -A6 "Allocated resources"
```

### Tolerations scope

`scheduling.tolerations` applies to the platform Deployments (`apigateway`, `auth`, `builder`, `cwcback`, `datasynchub`, `emailbuilder`, `frontendoneconnect`, `logs`, `metrics`, `otel-collector`) and to the workspace pods the Builder creates.

> 💡 Keep at least one node pool without taints for the database (`mysql-0`) and the setup jobs (`mysql-config`, `frontend-apigateway-sync`). If `mysql-0` is `Pending` with `had untolerated taint` while the other pods schedule, add an untainted node pool.

---

## 4. PersistentVolumeClaim Problems

MariaDB, DataSyncHub, and the Kafka broker all use persistent storage. Storage issues surface as pods stuck in `Pending` or failing to mount.

```bash
kubectl get pvc -A
kubectl describe pvc -n oneconnect data-mysql-0
kubectl describe pvc -n datasynchub st-datasynchub
kubectl get storageclass
```

### PVCs `Pending`: the storage class does not exist in the cluster

**Symptom** — `data-mysql-0` (namespace `oneconnect`), `st-datasynchub` (namespace `datasynchub`), and/or `data-0-kafka-cluster-main-pool-0` (namespace `kafka`) stay `Pending`. `kubectl describe pvc` shows `storageclass.storage.k8s.io "<name>" not found`, and the pods report `pod has unbound immediate PersistentVolumeClaims`.

**Cause** — Each cloud values file names a storage class, and the chart uses that name as-is. The class must exist in your cluster.

| Values file | `mysql.persistence.storageClass` | `datasynchub.persistence.storageClassName` | Kafka (`kafka-platform`) `nodePool.storage.class` |
|---|---|---|---|
| `values-aws.yaml` | `gp3` | `gp3` | `gp3` |
| `values-azure.yaml` | `managed-premium` | `managed-premium` | `managed-premium` |
| `values-gcp.yaml` | `pd-ssd` | `pd-ssd` | `pd-ssd` |

**Confirm** — compare the names above with the classes available in your cluster:

```bash
kubectl get storageclass
```

> 📌 Common cases: a new EKS cluster may offer only `gp2` until a `gp3` class is created, and a GKE cluster offers `standard`, `standard-rwo`, and `premium-rwo` by default.

**Fix** — either create a storage class with the expected name, or override the names at install time with classes that exist:

```bash
helm install oneconnect . \
  --namespace oneconnect --create-namespace \
  --values values-aws.yaml \
  --values secrets.local.yaml \
  --set dockerHub.token=dckr_pat_xxxxxxxxxxxx \
  --set networking.loadBalancer.type=external \
  --set mysql.persistence.storageClass=gp2 \
  --set datasynchub.persistence.storageClassName=gp2 \
  --timeout 15m

# Kafka broker storage (phase 2)
helm install kafka-platform ./kafka-platform \
  --values kafka-platform/values.yaml \
  --values kafka-platform/values-aws.yaml \
  --values kafka-platform/secrets.local.yaml \
  --set nodePool.storage.class=gp2 \
  --set loadBalancer.type=external
```

> ⚠️ Set `datasynchub.persistence.storageClassName` to a real class name rather than an empty string. Storage class settings on an existing PVC cannot be changed by `helm upgrade`; if a PVC was already created with an unavailable class, delete the `Pending` PVC (it holds no data yet) and run the upgrade with the corrected value.

### Other storage symptoms

| Symptom | Cause | Fix |
|---|---|---|
| PVC stuck `Pending`, no events, `mysql.persistence.storageClass` empty | No default StorageClass in the cluster | Set `mysql.persistence.storageClass` to a class that exists, or mark one as default |
| `provisioning failed` / `waiting for a volume to be created, either by external provisioner` | CSI driver not installed | Install the cloud CSI driver: EBS (EKS), Azure Disk (AKS), PD (GKE) |
| PVC `Bound` but pod fails to mount | Zone mismatch between volume and node | Use a topology-aware (`WaitForFirstConsumer`) StorageClass |
| PVC `Pending` with `exceeded quota` | Namespace storage quota reached | Raise the quota, or lower `mysql.persistence.size` / `datasynchub.persistence.size` |

---

## 5. A SAP Link Was Created, But No Producer Pod Appears

The Producer and Consumer are not part of the Helm install. The Builder creates them through the Kubernetes API when you create a SAP Link, so the Helm-side check is that the Builder has the permissions it needs.

**Confirm** — check the Builder logs and its permissions:

```bash
kubectl logs -n oneconnect -l app=builder --tail=200 | grep -E 'ERROR sending request|forbidden'

kubectl auth can-i create deployments \
  --as=system:serviceaccount:oneconnect:builder-service-account \
  -n oneconnect
```

A `no` answer, or `forbidden` in the Builder logs, means the Builder's RBAC is not in place.

**Fix** — keep `rbac.create: true` and `services.builder.serviceAccount.create: true` (the chart defaults) in your values, then run `helm upgrade` with your usual flags ([§8](#8-upgrades)). This creates the `builder-service-account` ServiceAccount, the `builder-role` Role, and its RoleBinding.

> 💡 For the full diagnosis (Builder log messages, workspace limits, image tags, and the other reasons a SAP Link may have no pods), see [06 §8 — The SAP Link Appears in the UI, but Its Pods Are Not Running](./06-No_Data_Reaching_Destination.md#8-the-sap-link-appears-in-the-ui-but-its-pods-are-not-running).

---

## 6. MariaDB Will Not Start, or Services Cannot Reach It

Five services wait for MariaDB through a `wait-for-mysql` init container before starting: **`auth`, `builder`, `cwcback`, `logs`, and `metrics`**. The other workloads (`apigateway`, `emailbuilder`, `frontendoneconnect`, `datasynchub`, `otel-collector`) start independently of MariaDB.

The databases are created the first time MariaDB starts on an empty volume. The database setup Job, **`<release>-mysql-config`** (for example `oneconnect-mysql-config`), then grants the application user (`mysql.user`, stored in `mysql-secret`) access to the service databases.

> ℹ️ The database server is **MariaDB 12.3 LTS**. The Kubernetes objects keep their historical `mysql` names (`mysql-0`, `mysql-secret`, the `mysql-config` Job, the `wait-for-mysql` init container and the `mysql.*` chart values), and the log messages printed by those scripts still say "MySQL". The MariaDB images do not include the `mysql` or `mysqladmin` commands; use `mariadb` and `mariadb-admin` instead:
>
> ```bash
> kubectl exec -it mysql-0 -n oneconnect -- mariadb -uroot -p
> ```

```bash
kubectl get pods -n oneconnect -l app=mysql
kubectl logs  -n oneconnect -l app=mysql --tail=100
kubectl get jobs -n oneconnect
kubectl logs  -n oneconnect -l app=mysql-config
```

### `auth` works, but `builder` / `cwcback` / `logs` / `metrics` report `Access denied`

**Symptom** — `auth` is `Running`, while `builder`, `cwcback`, `logs`, or `metrics` pass their init container and then go into `CrashLoopBackOff`. Their logs show `Access denied for user '<mysql.user>'@'%' to database '<database>'`.

**Cause** — the database setup Job did not complete, for example because MariaDB was still starting or its PVC was `Pending`.

**Confirm:**

```bash
kubectl get jobs -n oneconnect
# A completed Job shows COMPLETIONS 1/1

kubectl logs -n oneconnect -l app=mysql-config
# Success ends with:
#   ✓ Granted privileges on: <databases>
#   === Configuration Complete ===
# A timeout shows:
#   ERROR: MySQL did not become ready in time
```

**Fix** — re-run the database setup:

#### Re-run the database setup

1. Make sure `mysql-0` is `1/1 Running`.
2. Delete the setup Job:
   ```bash
   kubectl delete job oneconnect-mysql-config -n oneconnect
   ```
3. Run `helm upgrade` with the same values file and flags you used at install ([§8](#8-upgrades)).
4. Wait for `kubectl get jobs -n oneconnect` to show `1/1`, then restart the affected services:
   ```bash
   kubectl rollout restart deployment builder cwcback logs metrics -n oneconnect
   ```

### Other MariaDB symptoms

While MariaDB is not ready, the `wait-for-mysql` init container prints `ERROR: MySQL user not ready after 300 seconds` together with the checks shown above, and continues once MariaDB is ready.

| Symptom | Cause | Fix |
|---|---|---|
| MariaDB pod `Pending` | PVC unbound | See [§4](#4-persistentvolumeclaim-problems) |
| `auth`/`builder`/`cwcback`/`logs`/`metrics` in `Init:0/1` → `Init:Error` while MariaDB is not yet ready | MariaDB still starting | Wait; the init containers retry and continue once MariaDB is `1/1` |
| Init containers keep failing and the `mysql-config` log shows `ERROR: MySQL did not become ready in time` while `mysql-0` is `Running`, after a reinstall | The `data-mysql-0` volume from a previous install was kept and holds the passwords from that install | Install with the same `mysql.rootPassword` / `mysql.password` as the original install, or (if the data is not needed) delete PVC `data-mysql-0` before reinstalling |
| `Access denied for user` in the `auth` logs | `mysql-secret` does not match the credentials stored in the database, for example because `secrets.local.yaml` changed after the first install | Confirm the values in `mysql-secret`, or set the password in the database to match |
| MariaDB restarts repeatedly (`OOMKilled`) | Memory limit too low | Raise `mysql.resources.limits.memory` |
| MariaDB restarts with `Liveness probe failed` during its first start | First-time initialization on a slow disk | Raise `mysql.livenessProbe.initialDelaySeconds` |
| `mysql-0` in `CrashLoopBackOff` after moving from an older chart; the log shows InnoDB or data-dictionary errors | The `data-mysql-0` volume was created by MySQL 8, which MariaDB cannot open | Install in a new namespace (or delete the old PVC, which erases its data). To keep the data, export it from the old installation with a dump and import it into the new one |

> 📌 Keep `mysql.port` at `3306`; the in-cluster clients connect on that port.

---

## 7. No External IP / Cannot Reach the UI

```bash
kubectl get svc -n oneconnect
```

The chart creates two load balancers: `internal-frontend` (the UI, port **5050**) and `internal-apigateway` (port **9000**). Open the UI at:

```
http://<EXTERNAL-IP of internal-frontend>:5050
```

| Symptom | Cause | Fix |
|---|---|---|
| `EXTERNAL-IP` stays `<pending>` | No cloud LoadBalancer controller, quota reached, or (EKS) subnets not tagged for load balancers | Install the cloud provider's load balancer controller; check `kubectl get events -n oneconnect` for `SyncLoadBalancerFailed` |
| IP assigned but unreachable from the internet | LoadBalancer provisioned as internal | Install/upgrade with `--set networking.loadBalancer.type=external`, as the install guides do |
| Reachable from the VPC only — intended | `networking.loadBalancer.type: internal` (the values default) | Connect from inside the network or via VPN, or use `external` |
| IP reachable, page does not load | The UI listens on port `5050` | Use `:5050`, not `:80` |
| UI loads, but login or API calls do not reach the API Gateway | The `frontend-apigateway-sync` hook has not yet set the frontend's API address | See [§9](#9-helm-reports-failed-post-install-timed-out-waiting-for-the-condition) |

```bash
kubectl describe svc -n oneconnect internal-frontend | grep -A10 Annotations
kubectl get events -n oneconnect --sort-by=.lastTimestamp | tail -20
```

The `internal` setting adds a cloud-specific annotation (`aws-load-balancer-internal`, `azure-load-balancer-internal`, or `networking.gke.io/load-balancer-type`). Switching to `external` removes it. The annotation is chosen from `cloudProvider`, which is set by the cloud values file — always include `--values values-<cloud>.yaml` so the annotation for your cloud is applied.

> 💡 Control internal vs. external through `networking.loadBalancer.type` only. The chart manages the internal-load-balancer annotations itself, so adding them under `networking.loadBalancer.annotations` has no effect.

---

## 8. Upgrades

**Pass the same values file and flags on every upgrade.** We recommend keeping your install values in your own values file, so every upgrade uses exactly the same settings.

```bash
helm upgrade oneconnect . \
  --namespace oneconnect \
  --values values-aws.yaml \
  --values secrets.local.yaml \
  --set dockerHub.token=dckr_pat_xxxxxxxxxxxx \
  --set networking.loadBalancer.type=external \
  --timeout 15m
```

| Point to check | Detail |
|---|---|
| Include `--set dockerHub.token` | The secret is rebuilt on each upgrade, so pass the token every time; otherwise new pods go into `ImagePullBackOff` ([§2](#2-imagepullbackoff--errimagepull)) |
| Include `--set networking.loadBalancer.type=external` | The install guides use this flag. Keep `networking.loadBalancer.type` on every upgrade so the UI and API Gateway load balancers stay public |
| Include `--values secrets.local.yaml` | The chart requires the secrets on every upgrade; use the **same** values as the first install, because the database keeps the passwords it was created with |
| Include `--values values-<cloud>.yaml` | Keeps the cloud-specific settings (`cloudProvider`, storage classes, annotations) applied |
| Include any other `--set` you used | For example storage class overrides ([§4](#4-persistentvolumeclaim-problems)) or `scheduling.tolerations` |
| Include `--timeout 15m` | The post-install/post-upgrade hook runs again on every upgrade ([§9](#9-helm-reports-failed-post-install-timed-out-waiting-for-the-condition)) |
| New image tag not taking effect | `imagePullPolicy: IfNotPresent` — force with `kubectl rollout restart deployment -n oneconnect` |
| Existing Producer/Consumer not upgraded | They are created from the Builder's `DOCKER_IMAGE_VERSION_*` values at SAP Link creation time; existing ones keep their original tag until recreated |

> 💡 Keep your overrides in your own values file (for example `my-values.yaml`) and pass it after the cloud file on both install and upgrade: `--values values-aws.yaml --values my-values.yaml`.

To review the release revisions:

```bash
helm history oneconnect -n oneconnect
```

| Message during upgrade | Meaning | Fix |
|---|---|---|
| `Job.batch "oneconnect-mysql-config" is invalid: spec.template: ... field is immutable` | The database setup Job from the previous revision is still present | [Re-run the database setup](#re-run-the-database-setup) |
| `StatefulSet.apps "mysql" is invalid: spec: Forbidden: updates to statefulset spec for fields other than ...` or `PersistentVolumeClaim "st-datasynchub" is invalid: spec: Forbidden: spec is immutable after creation` | `mysql.persistence.*` or `datasynchub.persistence.storageClassName` differs from the installed value | Use the same storage values as the original install |

---

## 9. Helm Reports `failed post-install: timed out waiting for the condition`

**Symptom** — `helm install` (or `helm upgrade`) ends with `Error: INSTALLATION FAILED: failed post-install: timed out waiting for the condition` (or `failed post-upgrade`), and `helm list` shows the release as `failed`. The pods themselves are often healthy.

**Cause** — After installing, the chart runs the `<release>-frontend-apigateway-sync` hook Job, which waits for the `internal-apigateway` load balancer address and then configures the frontend's `NEXT_PUBLIC_API_BASE`. Install and upgrade with `--timeout 15m`, as the install guides do.

**Confirm:**

```bash
kubectl get pods -n oneconnect
kubectl get jobs -n oneconnect | grep frontend-apigateway-sync
kubectl logs -n oneconnect -l app=frontend-apigateway-sync
kubectl get svc internal-apigateway -n oneconnect
```

| Hook log | Meaning |
|---|---|
| `ApiGateway endpoint is not ready yet. Retrying in 10s...` | Still waiting for the load balancer address — normal for the first minutes |
| `Frontend configured successfully with NEXT_PUBLIC_API_BASE=http://...:9000` | The hook finished; the UI is ready to use |
| `ERROR: ApiGateway LoadBalancer endpoint was not assigned after 600s.` | The cloud has not assigned an address to `internal-apigateway`; see [§7](#7-no-external-ip--cannot-reach-the-ui) |

**Fix:**

1. If the pods are healthy and only the timeout was too short, run `helm upgrade` with your install flags and `--timeout 15m` ([§8](#8-upgrades)). The hook runs again and marks the release `deployed`.
2. If the hook reported `ERROR: ApiGateway LoadBalancer endpoint was not assigned`, resolve the load balancer first ([§7](#7-no-external-ip--cannot-reach-the-ui)), then run the upgrade.
3. If the hook pod itself is in `ErrImagePull`, set `networking.loadBalancer.autoConfigureFrontend.kubectlImage` to a `kubectl` image your cluster can pull, then upgrade.

> 💡 To check the address the frontend is using: `kubectl get deploy frontendoneconnect -n oneconnect -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="NEXT_PUBLIC_API_BASE")].value}'`. After a successful hook it is `http://<internal-apigateway address>:9000`.

---

## Summary

1. Give the stack **15 minutes** and always install/upgrade with `--timeout 15m` — 300-second readiness delays and the post-install hook are normal.
2. Install in order: **Strimzi → kafka-platform → oneconnect**, gating on pod readiness at each phase.
3. The chart has **no built-in credentials**: create `secrets.local.yaml` (and `kafka-platform/secrets.local.yaml`) before installing, pass them on every install and upgrade, and keep the same values afterwards.
4. `ImagePullBackOff` is nearly always the **Docker Hub token** — always pass `dockerHub.token`, on install and on every `helm upgrade`.
5. **Pass the same values file and flags on every upgrade**, especially `--set networking.loadBalancer.type=external` and the token — we recommend keeping them in your own values file.
6. `Pending` PVCs usually mean the **storage class** in the cloud values file does not exist in your cluster — compare with `kubectl get storageclass` and override if needed.
7. If `auth` works but `builder`/`cwcback`/`logs`/`metrics` report `Access denied`, [re-run the database setup](#re-run-the-database-setup).
8. Missing Producer/Consumer pods after creating a SAP Link start with the **Builder's RBAC** — see [06 §8 — The SAP Link Appears in the UI, but Its Pods Are Not Running](./06-No_Data_Reaching_Destination.md#8-the-sap-link-appears-in-the-ui-but-its-pods-are-not-running) for the full diagnosis.
