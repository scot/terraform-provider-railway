# Fork patch: `environment_id` on `railway_service`

Forked from `terraform-community-providers/terraform-provider-railway` (v0.6.2, latest
upstream release as of April 2026).

## Why

Upstream `railway_service` has no way to target a non-default environment. `Create()`
always builds `ServiceCreateInput` with only `Name`/`ProjectId`, and the follow-up
`updateServiceInstance` mutation hardcodes `environmentId: null`. Both resolve, via
`defaultEnvironmentForProject()`, to the project's **oldest** environment — which is
not necessarily the environment you actually want a new service to run in if a
project has more than one (e.g. separate staging/production environments).

## What changed

Full diff: compare this branch against upstream `master`. Summary:

- Added `environment_id` (Optional, Computed, `RequiresReplace`) to the
  `railway_service` resource schema/model.
- `Create()`: resolves `environment_id` (explicit value, or falls back to
  `defaultEnvironmentForProject()` — unchanged upstream behavior — if unset), passes it
  to `ServiceCreateInput.EnvironmentId` (already a valid field in the generated API
  client — genqlient's annotation for it already existed upstream, just never
  populated) and to the now-patched `updateServiceInstance` call.
- `updateServiceInstance` (`generated.go` + `resource_service.graphql`): hand-edited to
  take `environmentId *string` as a real GraphQL variable instead of a hardcoded
  `null` literal. **This is the change that actually matters** — it's what scopes
  source/cron/start-command settings to a specific environment's instance.
- `getAndBuildServiceInstance()` (used by `Read()`, including right after
  `terraform import <id>`): if `environment_id` isn't already known, discovers it via
  `getServiceInstances` (enumerates the service's actual per-environment instances).
  Errors if a service has instances in more than one environment, asking you to set
  `environment_id` explicitly to disambiguate.
- `Update()`: passes the (immutable, `RequiresReplace`) stored `environment_id` through
  to `updateServiceInstance`.
- `ImportState()`: accepts an optional composite import ID
  `"<service_id>:<environment_id>"` (in addition to a plain `<service_id>`), seeding
  `environment_id` directly into state at import time instead of relying on
  `getAndBuildServiceInstance`'s discovery — needed for any pre-existing service that
  already has instances in more than one environment (discovered empirically: an
  already-imported service can have this even without ever being created fresh via
  this fork), where discovery would otherwise error asking you to disambiguate.

No `.graphql` schema regeneration was run — `ServiceCreateInput.EnvironmentId` already
existed in generated code (upstream just never set it), and the `updateServiceInstance`
change was small enough to hand-edit directly in `generated.go` to match the `.graphql`
source, rather than needing genqlient + a live schema introspection.

## What's NOT verified

This has not been tested against the live Railway API. In particular:

- Railway's own doc comment on `ServiceCreateInput.EnvironmentId` says: *"If the
  specified environment is a fork, the service will only be created in it. Otherwise
  it will [be] created in all environments that are not forks of other environments."*
  Since a regular named environment (e.g. "staging") is not a "fork" (PR-preview)
  environment, this suggests passing `environment_id` to `serviceCreate` may **not**
  prevent the service shell from also existing in other non-fork environments. What
  this patch reliably controls is the **instance settings** (source/cron/start
  command) via `updateServiceInstance` — if an unintended environment's instance never
  gets a source/cron configured, it has nothing to run, which is the actual safety
  property needed.
- **Test on one service before relying on this for a bulk creation.** Create one
  `railway_service` with `environment_id` set to the target environment's real ID,
  confirm in the Railway dashboard that (a) the source/cron/start-command landed in
  the intended environment, and (b) nothing unexpected deployed elsewhere, before
  applying this pattern at scale.

## Building

No goreleaser/signing set up in this fork — `.github/workflows/build-fork-binary.yml`
cross-compiles plain `go build` binaries (darwin_arm64, darwin_amd64, linux_amd64) and
uploads them as downloadable Actions artifacts. Or build locally:

```bash
GOOS=darwin GOARCH=arm64 go build -o terraform-provider-railway_v0.6.3-envid .
```

## Using it

Point Terraform at this build via a `filesystem_mirror` CLI config: provider source
becomes `<your-namespace>/railway`, version `0.6.3-envid`, pointing at the built
binary.
