# Deployment & Installation

Failures that happen while installing or upgrading the Helm chart, or that prevent pods from reaching `Running`.

Applies to the SmartGateway Helm chart on EKS, AKS, and GKE. For SAP BTP / Kyma, see the [BTP deployment troubleshooting](../Smart_Gateway/BTP/SmartGateway_BTP_Kyma_Deployment.md#troubleshooting) — Cloud Connector and Istio issues are covered there.

---

## 0. First, Rule Out "It Is Just Slow"

Before diagnosing anything, know the chart's normal timings:

| Behavior | Normal value | Implication |
|---|---|---|
| Readiness probe initial delay | **300 seconds** on most services | A pod showing `0/1` for the first ~5 minutes is expected |
| MySQL wait init container | Up to **300 seconds** (60 retries × 5 s) | Dependent services legitimately sit in `Init:0/1` for minutes |
| MySQL readiness probe | **200 seconds** initial delay | MySQL is the long pole on a cold install |
| Recommended install timeout | `--timeout 15m` | Shorter timeouts produce false failures |

```bash
# Watch rather than re-running helm
kubectl get pods -n oneconnect -w
```

A pod is only a problem if it is `CrashLoopBackOff`, `ImagePullBackOff`, `Pending` beyond a few minutes, or still `0/1` well past its initial delay.

---

## 1. `helm install` Fails Immediately

### Missing required values

The chart requires the Docker Hub token to be supplied at install time — it ships empty.

```
Error: execution error at (oneconnect/templates/secrets.yaml): ...
```

Re-run with every required flag:

```bash
helm install oneconnect . \
  --namespace oneconnect --create-namespace \
  --values values-aws.yaml \
  --set dockerHub.token=dckr_pat_xxxxxxxxxxxx \
  --set networking.loadBalancer.type=external \
  --timeout 15m
```

### Render the templates without installing

The fastest way to see what Helm is actually producing:

```bash
helm template oneconnect . --values values-aws.yaml \
  --set dockerHub.token=dummy > /tmp/rendered.yaml

helm lint . --values values-aws.yaml --set dockerHub.token=dummy
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

### Deployment order was not respected

The stack is installed in three phases. Installing OneConnect before Kafka is available produces pods that start without a cluster to connect to.

| Phase | Component | Gate before proceeding |
|---|---|---|
| 1 | Strimzi operator | `strimzi-cluster-operator` pod shows `1/1 Running` |
| 2 | `kafka-platform` chart | Kafka, Schema Registry, and AKHQ pods all `1/1 Running` |
| 3 | `oneconnect` chart | — |

```bash
kubectl get pods -n kafka | grep strimzi-cluster-operator
kubectl get pods -n kafka
```

> 📌 Wait for the operator to reach `1/1 Running` before starting phase 2. The Kafka custom resources are accepted by the API server but only reconciled once the operator is ready, so starting early surfaces later as "Kafka pods not starting".

---

## 2. `ImagePullBackOff` / `ErrImagePull`

Usually related to the Docker Hub token.

### Confirm it

```bash
kubectl describe pod -n oneconnect <pod-name> | tail -30
```

Look for `Failed to pull image ... unauthorized` or `pull access denied`.

### How the secret works

The chart builds a `kubernetes.io/dockerconfigjson` secret named **`docker-hub-credentials`** from three values:

| Value | Default | Notes |
|---|---|---|
| `dockerHub.registry` | `https://index.docker.io/v1/` | Rarely changed |
| `dockerHub.username` | `onibexenjoy` | The Onibex Docker Hub account |
| `dockerHub.token` | *(empty)* | **You must supply this** |
| `dockerHub.createSecret` | `true` | Set to `false` only if you manage the secret yourself |

When `createSecret` is enabled the same secret is created in **three namespaces** — `oneconnect`, `datasynchub`, and `kafka` — because workloads in all three pull Onibex images.

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

### Fixes

| Cause | Fix |
|---|---|
| Token not supplied | Re-run `helm upgrade` with `--set dockerHub.token=...` |
| Token expired or was revoked | Generate a new Docker Hub PAT, then `helm upgrade` with the new value |
| Token pasted with quotes or whitespace | Re-set it cleanly; shell quoting frequently corrupts `dckr_pat_` strings |
| `createSecret: false` but no secret provided | Either set it to `true`, or create `docker-hub-credentials` manually in all three namespaces |
| Secret exists but pods still fail | Delete the pods so they retry: `kubectl delete pod -n oneconnect <pod>` |

After updating the token, the secret changes but running pods do not re-pull. Restart them:

```bash
kubectl rollout restart deployment -n oneconnect
```

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
| `Insufficient cpu` / `Insufficient memory` | Node pool too small | Scale the node group, or lower the `resources.requests` in values |
| `pod has unbound immediate PersistentVolumeClaims` | No storage provisioner / wrong storage class | See [§4](#4-persistentvolumeclaim-problems) |
| `had untolerated taint` | Nodes are tainted | Add matching entries under `scheduling.tolerations` in values |
| `didn't match node affinity` | `scheduling.nodeAffinity` rules exclude all nodes | Correct or disable the affinity rules |

Check capacity:

```bash
kubectl get nodes
kubectl describe nodes | grep -A6 "Allocated resources"
```


---

## 4. PersistentVolumeClaim Problems

MySQL and DataSyncHub both require persistent storage. Storage issues surface as pods stuck in `Pending` or failing to mount.

```bash
kubectl get pvc -A
kubectl describe pvc -n oneconnect data-mysql-0
kubectl get storageclass
```

| Symptom | Cause | Fix |
|---|---|---|
| PVC stuck `Pending`, no events | No default StorageClass | Set `mysql.persistence.storageClass` / `datasynchub.persistence.storageClassName` to a class that exists |
| `provisioning failed` | CSI driver not installed | Install the cloud CSI driver: EBS (EKS), Azure Disk (AKS), PD (GKE) |
| PVC `Bound` but pod fails to mount | Zone mismatch between volume and node | Use a topology-aware (`WaitForFirstConsumer`) StorageClass |
| PVC `Pending` with `exceeded quota` | Namespace storage quota reached | Raise the quota, or lower `persistence.size` |

---

## 5. A SAP Link Was Created, But No Producer Pod Appears

The Producer and Consumer are not part of the Helm install. The Builder creates them through the Kubernetes API when you create a SAP Link, so check the Builder first.

### Check the Builder first

```bash
kubectl logs -n oneconnect -l app=builder --tail=200 | grep -Ei 'error|forbidden|denied|unauthor'
```

### Cause A — RBAC is missing or disabled

The Builder needs a Role allowing it to manage `deployments`, `services`, `secrets`, and `ingresses` in the `oneconnect` namespace, bound to its ServiceAccount.

```bash
kubectl get serviceaccount builder-service-account -n oneconnect
kubectl get role builder-role -n oneconnect
kubectl get rolebinding -n oneconnect

# Definitive test — does the Builder actually have the permission?
kubectl auth can-i create deployments \
  --as=system:serviceaccount:oneconnect:builder-service-account \
  -n oneconnect
```

A `no` answer, or a `Forbidden` line in the Builder logs, confirms it.

**Fix:** ensure `rbac.create: true` and `services.builder.serviceAccount.create: true` in your values, then `helm upgrade`.

### Cause B — the workspace limit was reached

`KUBERNETES_MAX_WORKSPACES` caps how many SAP Links can exist. Check the effective value:

```bash
kubectl exec -n oneconnect -l app=builder -- env | grep -E 'KUBERNETES_MAX_WORKSPACES|KUBERNETES_WORKSPACE_REPLICAS'
```

Raise it via `services.builder.env.KUBERNETES_MAX_WORKSPACES` and `helm upgrade`.

### Cause C — the image tag does not exist

The Builder creates the workloads using fixed image tags:

```bash
kubectl exec -n oneconnect -l app=builder -- env | grep DOCKER_IMAGE_VERSION
```

If a tag was overridden to a version that is not published, the Deployment is created but its pods fail to pull. Confirm with `kubectl get pods -n oneconnect | grep -i producer` — an `ImagePullBackOff` here points at the tag, not the token (see [§2](#2-imagepullbackoff--errimagepull)).

### Cause D — the Builder cannot reach its database

```bash
kubectl get pods -n oneconnect -l app=builder
kubectl logs -n oneconnect -l app=builder -c wait-for-mysql
```

If the init container is still looping, this is a MySQL problem — see [§6](#6-mysql-will-not-start-or-services-cannot-reach-it).

---

## 6. MySQL Will Not Start, or Services Cannot Reach It

Every platform service waits on MySQL through an init container before starting. If MySQL is unhealthy, the whole namespace stalls in `Init:0/1`.

```bash
kubectl get pods -n oneconnect -l app=mysql
kubectl logs  -n oneconnect -l app=mysql --tail=100
kubectl get jobs -n oneconnect
kubectl logs  -n oneconnect -l app=mysql-config
```

The init container prints its own guidance after 300 seconds; these are the same four checks.

| Symptom | Cause | Fix |
|---|---|---|
| MySQL pod `Pending` | PVC unbound | See [§4](#4-persistentvolumeclaim-problems) |
| MySQL `Running`, init container still looping | Config job did not run or failed | Check `mysql-config` job logs; the databases/grants may not have been created |
| `Access denied for user` | `mysql-secret` does not match the credentials in the database | Confirm the values in `mysql-secret`, or set the password in the database to match |
| MySQL restarts repeatedly | Memory limit too low | Raise `mysql.resources.limits.memory` |

---

## 7. No External IP / Cannot Reach the UI

```bash
kubectl get svc -n oneconnect
```

Look for the frontend service and its `EXTERNAL-IP`. The UI listens on port **5050**:

```
http://<EXTERNAL-IP>:5050
```

| Symptom | Cause | Fix |
|---|---|---|
| `EXTERNAL-IP` stays `<pending>` | No cloud LoadBalancer controller | Install the cloud provider's load balancer controller |
| IP assigned but unreachable from the internet | LoadBalancer provisioned as internal | Re-install/upgrade with `--set networking.loadBalancer.type=external` |
| Reachable from the VPC only — intended | `type: internal` is the chart default | This is correct behavior; connect from inside the network or via VPN |
| IP reachable, page does not load | Wrong port | Use `:5050`, not `:80` |

```bash
kubectl describe svc -n oneconnect <frontend-service> | grep -A10 Annotations
kubectl get events -n oneconnect --sort-by=.lastTimestamp | tail -20
```

The `internal` setting adds a cloud-specific annotation (`aws-load-balancer-internal`, `azure-load-balancer-internal`, or `networking.gke.io/load-balancer-type`). Switching to `external` removes it. Verify `cloudProvider` in your values matches the cluster you are on, so the annotation for your cloud is applied.

---

## 8. Upgrades

```bash
helm upgrade oneconnect . \
  --namespace oneconnect \
  --values values-aws.yaml \
  --set dockerHub.token=dckr_pat_xxxxxxxxxxxx
```

| Point to check | Detail |
|---|---|
| Include `--set dockerHub.token` | The secret is rebuilt on each upgrade, so pass the token every time |
| Include `--values values-<cloud>.yaml` | Keeps the cloud-specific settings applied |
| New image tag not taking effect | `imagePullPolicy: IfNotPresent` — force with `kubectl rollout restart deployment -n oneconnect` |
| Existing Producer/Consumer not upgraded | They are created from the Builder's `DOCKER_IMAGE_VERSION_*` values at SAP Link creation time; existing ones keep their original tag until recreated |

Always confirm what changed:

```bash
helm history oneconnect -n oneconnect
helm get values oneconnect -n oneconnect
```

---

## Summary

1. Give the stack **15 minutes** — 300-second readiness delays are normal, not failures.
2. Install in order: **Strimzi → kafka-platform → oneconnect**, gating on pod readiness at each phase.
3. `ImagePullBackOff` is nearly always the **Docker Hub token**; remember it is needed on every `helm upgrade` too, and that Producer/Consumer pods expose an expired token long after install.
4. Missing Producer/Consumer pods after creating a SAP Link means the **Builder** was blocked — check RBAC, the workspace limit, and image tags.
5. Storage issues show up as `Pending` pods — check the StorageClass and the cloud CSI driver.
