# argocd-fun

GitOps content for the Argo CD lab in [deploy-infra](../deploy-infra), using the
[app of apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/) pattern.

```
bootstrap/root.yaml   the root Application; applied once by hand, points at apps/
apps/                 one Argo CD Application per file; the root syncs this directory
appsets/              ApplicationSets; synced onto the hub by apps/appsets.yaml, so also under the root
workloads/            plain manifests deployed by the sets above (currently guestbook with a smoke test)
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

## Progressive rollout with a smoke-test gate

`appsets/guestbook-rollout.yaml` rolls `workloads/guestbook` through the spokes in waves using
`strategy.type: RollingSync`: first every cluster whose Secret has `env=dev`, then `env=prod`.
A wave only starts when every Application in the previous wave is Healthy with a succeeded sync,
and `workloads/guestbook/smoke-test.yaml` is a `PostSync` hook Job that curls the Service, so a
failing smoke test on dev holds prod at the previous revision. The hub must run the
applicationset controller with progressive syncs enabled (deploy-infra: `VARIANT=progressive`).

Try it:

1. Commit a harmless change to `workloads/guestbook/` (a label, `replicas: 2`) and watch
   `kubectl -n argocd get appset guestbook-rollout -o yaml` under `status.applicationStatus`:
   dev goes Pending, Progressing, Healthy; only then does prod leave Waiting.
2. Change the path in `smoke-test.yaml` to one that 404s and push. Dev's sync fails on the hook,
   prod never starts, and `argocd app get guestbook-rollout-spoke-b` still shows the old revision.
   Revert the commit and both recover in order.

Argo CD polls git about every three minutes; `argocd app get <app> --hard-refresh` skips the wait.

## Conventions

- Every Application carries the `resources-finalizer.argocd.argoproj.io` finalizer so that
  pruning it also deletes what it deployed.
- Destinations use `name:` (the cluster Secret name), never a server URL.
- `syncPolicy.automated` with `prune` and `selfHeal` everywhere: git is the only input.
