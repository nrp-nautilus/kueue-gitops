# kueue-gitops

Argo CD source for [Kueue](https://kueue.sigs.k8s.io/) on the NRP Nautilus cluster.
The Argo CD Application is `argo/kueue`; it deploys into `kueue-system`.

## Layout

| Path | What |
|---|---|
| `upstream/<version>/manifests.yaml` | Upstream release manifest, vendored unmodified |
| `controller_manager_config.yaml` | NRP Kueue `Configuration` (replaces the upstream ConfigMap) |
| `patches/` | NRP changes to upstream objects (replicas, resources, namespace metadata) |
| `flowschema.yaml` | Sends Kueue's leader-election lease traffic to the `leader-election` API priority level |
| `kustomization.yaml` | Ties the above together; this is what Argo CD renders |
| `argocd/application.yaml` | The Argo CD Application, applied by hand |

## Safety

The Kueue pod webhook uses `failurePolicy: Fail` across nearly every namespace,
so Kueue being down blocks pod creation cluster-wide. Before merging a change:

```bash
kubectl kustomize . | kubectl diff --server-side --force-conflicts --field-manager=argocd-controller -f -
```

The namespace and CRDs carry `argocd.argoproj.io/sync-options: Delete=false`:
deleting the Application, or pruning, will never delete them. Deleting the CRDs
would delete every ClusterQueue, LocalQueue and Workload.

Never set `spec.conversion.strategy: None` on the Kueue CRDs. At startup Kueue
injects its CA into the conversion webhook of `clusterqueues`, `cohorts`,
`localqueues`, `multikueueclusters` and `workloads`, and it will not start if
`conversion.webhook.clientConfig` is missing. Running pods keep working, so the
break only shows on the next restart. On 2026-11-24 a manual patch did this;
when kured later rebooted the node under the last healthy replica, pod creation
was blocked cluster-wide for 1h40m. Fix conversion problems here instead (see the
v1beta1 patch in `kustomization.yaml`).

## Changing the configuration

Edit `controller_manager_config.yaml`, sync, then restart the controller
(Kueue reads its config only at startup):

```bash
kubectl rollout restart -n kueue-system deploy/kueue-controller-manager
```

## Upgrading Kueue

1. Download the new release manifest into `upstream/<version>/manifests.yaml`.
2. Point `kustomization.yaml` at it.
3. Diff the new release's default config against `controller_manager_config.yaml`
   and read the release notes for every minor version in between.
4. Run the diff above, then sync. If the release drops an object (v0.20 dropped
   the `v1beta1.visibility.kueue.x-k8s.io` APIService), prune it in that sync:
   a stale APIService with no backend breaks API discovery and blocks
   namespace deletion.
5. On the upgrade to v0.21 or later, delete the v1beta1-removal patch from
   `kustomization.yaml`; upstream no longer ships v1beta1, so its `test` op
   fails the build until you do.
