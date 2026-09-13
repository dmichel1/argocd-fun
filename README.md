# argocd-fun

GitOps content for the Argo CD lab in [deploy-infra](../deploy-infra), using the
[app of apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/) pattern.

```
bootstrap/root.yaml   the root Application; applied once by hand, points at apps/
apps/                 one Argo CD Application per file; the root syncs this directory
```

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

## Conventions

- Every Application carries the `resources-finalizer.argocd.argoproj.io` finalizer so that
  pruning it also deletes what it deployed.
- Destinations use `name:` (the cluster Secret name), never a server URL.
- `syncPolicy.automated` with `prune` and `selfHeal` everywhere: git is the only input.
