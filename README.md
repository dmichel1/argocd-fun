# argocd-fun

GitOps content for the Argo CD lab in [deploy-infra](../deploy-infra), using the
[app of apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/) pattern.

```
bootstrap/root.yaml   the root Application; applied once by hand, points at apps/
apps/                 one Argo CD Application per file; the root syncs this directory
appsets/              ApplicationSets; synced onto the hub by apps/appsets.yaml, so also under the root
platform/             cluster components deployed to every spoke (Argo Rollouts via OLM)
workloads/            what runs on the spokes (guestbook as a blue-green Rollout with a smoke test)
```

Two patterns compose here. `apps/` is the
[app of apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/):
every Application is written by hand. One of those Applications, `apps/appsets.yaml`, syncs
`appsets/`, which holds
[ApplicationSets](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/):
a template plus a generator stamps further Applications out. Everything descends from the root.

## How it works

1. `deploy-infra` builds a hub cluster with Argo CD and registers spoke clusters as
   cluster Secrets named `spoke-a`, `spoke-b`, ...
2. `make bootstrap` in deploy-infra applies `bootstrap/root.yaml` to the hub.
3. The `root` Application watches `apps/` on `main`. Every file there is an Application
   that Argo CD creates, updates, or prunes to match git.
4. Each child Application deploys a workload to a destination cluster by name.

## Adding an app

Add `apps/<name>.yaml` containing an `Application` in namespace `argocd`, push to `main`,
and the root picks it up on its next poll (about three minutes) or immediately after
`argocd app get root --hard-refresh`. Delete the file to remove the app and its workloads.

## ApplicationSets

`appsets/helm-guestbook.yaml` uses the cluster generator with `matchLabels: role: spoke`, so it
produces `helm-guestbook-<spoke>` for every cluster Secret carrying that label. deploy-infra sets
the label when it registers a spoke. Register a third spoke and a third Application appears with
no commit here; remove the label and the Application (and its workloads, via the finalizer) go away.

The sets are synced by the `appsets` Application, which the root creates from `apps/appsets.yaml`.
To add a set, add a file under `appsets/`, list it in `appsets/kustomization.yaml`, and push.
Preview what a set would generate with `argocd appset generate appsets/helm-guestbook.yaml`.

Deleting `apps/appsets.yaml` from git cascades all the way down: the root prunes the `appsets`
Application, its finalizer deletes the ApplicationSets, the sets delete their generated
Applications, and those Applications' finalizers delete the workloads on the spokes.

## Platform: Argo Rollouts on every spoke, via OLM

`platform/argo-rollouts/` installs Argo Rollouts with the
[argo-rollouts-manager](https://github.com/argoproj-labs/argo-rollouts-manager) operator through
Operator Lifecycle Manager: a CatalogSource for the locally built catalog, a Subscription in
`operators` whose `config.env` permits a cluster-scoped instance and names its namespace, and one
cluster-scoped `RolloutManager` in `argo-rollouts` (sync wave 2, `SkipDryRunOnMissingResource`,
because its CRD arrives with the operator). `appsets/platform-argo-rollouts.yaml` generates one
Application per `role=spoke` cluster. deploy-infra provides the prerequisites: the local registry,
OLM on each spoke, and the operator, bundle and catalog images, which upstream does not publish.

`operator-rbac-fix.yaml` works around an upstream gap: the operator's CSV lacks three rules that the
Rollouts v1.9.0 ClusterRole it creates contains, and Kubernetes' escalation check rejects the grant.

## Progressive rollout with a smoke-test gate

`appsets/guestbook-rollout.yaml` rolls `workloads/guestbook` through the spokes in waves using
`strategy.type: RollingSync`: first every cluster whose Secret has `env=dev`, then `env=prod`.
The hub must run the applicationset controller with progressive syncs enabled (deploy-infra:
`VARIANT=progressive`). A wave completes when every Application in it is **Synced and Healthy**;
that is the whole gate, and the workload is built so the smoke test feeds into it:

- The guestbook is a **blue-green Argo Rollout**. New pods come up behind `guestbook-ui-preview`,
  the `smoke-test` AnalysisTemplate runs a Job (the `job` provider) against that preview Service as
  `prePromotionAnalysis`, and only on success does the active Service switch. On failure the Rollout
  aborts by itself: the preview ReplicaSet is scaled down and the previous version keeps serving.
- Argo CD's built-in Rollout health is Progressing during analysis, Degraded on abort, Healthy once
  promoted. The Application inherits it, so dev's smoke test holds prod with no extra plumbing.
- The ApplicationSet template stamps `${ARGOCD_APP_REVISION_SHORT}` onto every resource, so every
  commit makes the apps OutOfSync and walks the waves. The stamp does not reach the Rollout's pod
  template (kustomize only knows that path for built-in kinds), so a rollout, and with it the smoke
  test, happens when the pod template itself changes, as in a real release. Commits that change
  nothing in the pods pass through the waves with the Rollout still Healthy.
- The template keeps `syncPolicy.automated.prune: true` even though RollingSync disables autosync:
  the controller reads the prune flag from there before disabling it.

Why not a hook? The RollingSync controller (`applicationset/progressivesync/progressive_sync.go`)
never looks at sync operations or hook results, so a `PostSync` smoke test did not hold prod in
this lab, and a plain Job needed same-wave and Replace/Force tricks to work. The Rollout's own health
is the clean signal. Set `autoPromotionEnabled: false` for a manual gate: the Rollout pauses,
Argo CD reports Suspended, and the wave waits until someone runs `kubectl argo rollouts promote`.

Try it:

1. Commit a pod-template change under `workloads/guestbook/` (an annotation on the pod template is
   enough) and watch
   `kubectl -n argocd get appset guestbook-rollout -o yaml` under `status.applicationStatus`, or
   `kubectl argo rollouts get rollout guestbook-ui -n guestbook-rollout --context kind-spoke-a -w`:
   dev goes Pending, Progressing (analysis running), Healthy; only then does prod leave Waiting.
2. Change the analysis URL path in `rollout.yaml` to one that 404s, bump the pod-template annotation
   so a rollout starts, and push. Dev's Rollout aborts
   and goes Degraded, prod stays Waiting, and `argocd app get guestbook-rollout-spoke-b` still shows
   the previous revision. Revert and both recover in order.

Argo CD polls git about every three minutes; `argocd app get <app> --hard-refresh` skips the wait.

## Conventions

- Every Application carries the `resources-finalizer.argocd.argoproj.io` finalizer so that
  pruning it also deletes what it deployed.
- Destinations use `name:` (the cluster Secret name), never a server URL.
- `syncPolicy.automated` with `prune` and `selfHeal` everywhere: git is the only input.
