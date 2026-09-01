# Delivery

How a service moves from a commit to a running process: branches and environments, the CI that gates them, the image build, per-commit publishing, and deployment. This file owns the pipeline; environment.md owns what the deployed system looks like once it is running, and says only that images "come from delivery".

## Contents

- Branches are environments
- The workflow files
- The tests job
- The release image
- Publishing per commit
- The registry
- The shared actions repository
- Deploying to the platform
- Platform configuration and its traps
- Migrations in the pipeline
- Dependency updates
- Retention

## Branches are environments

Two long-lived branches, each bound to one environment:

- **`develop` is staging.** Every commit publishes an image and deploys it — tests, image, migrations, serve, unattended. Staging is where push-and-see-it-running lives.
- **`main` is production.** Every commit publishes an image; the deploy steps exist only once a production platform does. Promotion is a merge from develop to main — after the first sync, never cherry-picks, so the histories stay converged and script or workflow fixes ride the same merge as code.

Pull requests run the tests job regardless of target branch. Deployment steps are mandatory, not gated: a missing secret or variable fails the pipeline loudly through the action's own environment checks rather than skipping the deploy and reporting green. A pipeline that silently does less than it claims is worse than a red one.

Versioning differs by what a repository is *for*. Services carry no SemVer tags and no releases: the commit is the version, the SHA is the deploy identifier. SemVer, API-breakage checks, and label-driven release automation belong to the shared contract packages — protos and identity — whose consumers resolve version ranges (see *Contracts and compatibility* in distributed-systems.md).

## The workflow files

Each service repository carries four small workflows and a dependabot configuration:

| File | Trigger | Does |
| --- | --- | --- |
| `pull_request.yml` | every PR | the tests job |
| `develop.yml` | push to develop | tests → publish → migrate → deploy (staging) |
| `main.yml` | push to main | tests → publish (production deploy steps join when the platform exists) |
| `cleanup-images.yml` | weekly cron | prunes the registry to a recent window |
| `dependabot.yml` | weekly | Swift and Actions bumps as PRs against develop |

Third-party actions are pinned to commit SHAs with a version comment; dependabot keeps the pins moving. The organization's own actions are pinned by SemVer tag (below).

## The tests job

CI reuses `swiftlang/github-workflows`' `swift_package_test.yml`, pinned to a release tag, collapsed from its default sweep to the one cell that matches production:

```yaml
jobs:
  tests:
    name: Tests
    uses: swiftlang/github-workflows/.github/workflows/swift_package_test.yml@<tag>
    with:
      linux_swift_versions: '["<toolchain>"]'
      linux_os_versions: '["<os>"]'
      linux_host_archs: '["<arch>"]'          # the arch production runs; never test what nothing ships
      linux_build_command: "swift test --disable-automatic-resolution"
      enable_windows_checks: false             # defaults to true; refuse the surprise
```

`--disable-automatic-resolution` makes the committed `Package.resolved` the build: CI fails when the manifest and the pin drift instead of silently resolving something newer. The macOS/iOS jobs run on runners only the swiftlang organization has — leave them off. Two accepted trades come with the reuse: no dependency caching (every run resolves and compiles cold; the price of not maintaining the workflow) and no static-SDK job (the published image links glibc, below, so a musl check guards nothing that ships — and vendored C++ in the protobuf toolchain has broken musl builds outright).

## The release image

Published images build from a `Containerfile`, two stages, glibc:

- **Build stage** on the Swift toolchain image: `COPY ./Package.*` and `swift package resolve` as their own layer so dependency resolution caches while manifests are unchanged, then `swift build --configuration release --static-swift-stdlib --product <service>`, then stage the binary, `swift-backtrace-static`, and every `*.resources` bundle.
- **Runtime stage** on the matching minimal OS image: `ca-certificates` and `tzdata` only, an unprivileged system user with `/app` as home, the staged files copied in with that owner, `SWIFT_BACKTRACE` configured, `ENTRYPOINT ["./<service>"]`.

A `.dockerignore` beside it excludes version control, `.github`, build state, secrets patterns, and everything not needed to compile the package.

Build natively for the deployment host's architecture — an ARM host means an ARM runner and `platforms: linux/arm64` — never under emulation, which turns a release build into an hour. The static-musl container plugin remains the local `Makefile` path (environment.md); the Containerfile is the delivery path, and the two are allowed to differ because only one of them publishes.

## Publishing per commit

The publish job runs after tests on every push to a deployment branch: buildx builds the Containerfile with the GitHub Actions layer cache and pushes two tags —

```
ghcr.io/<organization>/<organization>-<service>:<short-sha>
ghcr.io/<organization>/<organization>-<service>:<branch>
```

The **short SHA tag is immutable and is the only thing ever deployed**; rollback is deploying an older SHA. The moving branch tag (`develop`, `main`) exists for humans and bootstrap — pulling "the latest staging build" by hand, seeding a platform application before its first pipeline deploy — and nothing in the pipeline consumes it after that.

## The registry

The workflow's own `GITHUB_TOKEN` pushes, with `permissions: packages: write` on the job. One trap: a package that already exists — created by a manual push under a personal token — does not trust any repository's `GITHUB_TOKEN` until the package's *Manage Actions access* setting grants the repository write (admin, if the cleanup workflow is to delete versions). The setting is UI-only; the first pipeline push of every pre-existing package fails `permission_denied` until it is made.

## The shared actions repository

Deployment mechanics live in one organization repository — `<organization>/ci` — as composite actions, so a platform-API quirk is fixed once and reaches every service as a tag bump instead of hand-synced copies:

- **`dokploy-migrate`**: pins the migration application to the image, deploys the one-shot, polls the platform's deployment status until done; a failed or timed-out migration fails the pipeline before serve is touched.
- **`dokploy-deploy`**: pins the exact image tag on the application, then rolls it.

Both take the same inputs — `url`, `api-key`, `app-id`, `image` — and each action directory carries a same-named script the action step runs. Tags are plain SemVer without a `v` prefix, the same convention as the contract packages, and dependabot's `github-actions` ecosystem bumps consumers. The repository can stay private with organization-wide Actions access enabled.

The dividing line for what belongs there: **logic that changes** (a third-party platform's API mechanics) is centralized; **declarations that don't** (a cron line, a retention count, a package name) stay in each repository. The cleanup workflow, for instance, must live per-repo regardless — schedules only fire in the repository that hosts them, and package deletion needs the repository's own token.

## Deploying to the platform

The default platform is Dokploy, driven entirely over its HTTP API with an `x-api-key` token — the same two-call shape any platform needs: *update* the application's image reference to the exact SHA, then *deploy*. Pinning before rolling keeps the platform UI naming the commit that is actually running, and makes rollback "set the previous SHA and deploy".

The workflow binds the actions to an environment through secrets and variables — the endpoint and token as secrets, the application ids as variables, a `STAGING_` prefix distinguishing staging's from production's:

```yaml
      - name: Run staging migrations
        uses: <organization>/ci/dokploy-migrate@<tag>
        with:
          url: ${{ secrets.STAGING_DOKPLOY_URL }}
          api-key: ${{ secrets.STAGING_DOKPLOY_API_KEY }}
          app-id: ${{ vars.STAGING_<SERVICE>_MIGRATE_APP_ID }}
          image: ghcr.io/<organization>/<organization>-<service>:${{ steps.sha.outputs.short }}

      - name: Deploy to staging
        uses: <organization>/ci/dokploy-deploy@<tag>
        with:
          url: ${{ secrets.STAGING_DOKPLOY_URL }}
          api-key: ${{ secrets.STAGING_DOKPLOY_API_KEY }}
          app-id: ${{ vars.STAGING_<SERVICE>_APP_ID }}
          image: ghcr.io/<organization>/<organization>-<service>:${{ steps.sha.outputs.short }}
```

The platform API must be reachable from the runner: a public endpoint is two curls with a bearer key; a tailnet-only endpoint means a tailnet-join action in the workflow or publish-only CI with deploys triggered from inside. The API key is a deploy credential that can reconfigure every application — rotate it on any suspicion, and keep the platform account behind a second factor.

## Platform configuration and its traps

One application per process — `<service>`, `<service>-migrate`, `<service>-worker` — Docker-image sourced, environment carried per application (a platform's project-level variables are usually interpolation references, not injected values; prefer concrete values per app). Key files arrive as file mounts at the same paths the Compose environment used, so the service's configuration does not know which environment it is in.

Dokploy specifics, each learned the expensive way:

- **`command` replaces the image's entrypoint**, it does not append to it: write `./<service> database migrate`, never `database migrate`.
- **A one-shot application needs restart policy `none`**, or the platform's scheduler restarts the exited migration forever.
- **Application names are immutable and get a random suffix at creation.** The suffixed name is the internal DNS name — so every mTLS leaf needs the suffixed name as a SAN beside the plain service name (see *Transport security: the certificate volume* in environment.md), issued against the same CA.
- **Do not link an application to a registry entry**: on this platform that designates a *cluster* registry and re-uploads every image to it on deploy — which a read-only pull token cannot do. Registry credentials entered once in the platform UI land in the host's Docker login and cover pulls for every application.
- **A crash-looping service can pin a stale spec**: after fixing configuration, stop the application entirely, then deploy, rather than deploying over the loop.

## Migrations in the pipeline

Migrations run as their own application — the same image, `database migrate`, the owner's credentials — never folded into the serve entrypoint, which would hand the serving process credentials that can create roles (see *The service role* in persistence.md). The pipeline runs the migrate action unconditionally before every serve deploy: the migration library applies only the delta, so the common case is a fast no-op and ordering is guaranteed by construction rather than by remembering.

Two facts the ordering cannot fix, and one rule that absorbs both: the old serve build briefly runs against the new schema during every deploy, and a rollback runs old code against a schema that migrated forward — **so migrations are expand/contract**. Adding tables, nullable columns, and indexes is always safe; renames, drops, and tightening constraints ship in a later commit, after no deployed code references the old shape. A genuinely breaking migration is the rare event where the deploy is watched rather than unattended.

## Dependency updates

Dependabot targets develop (`target-branch: "develop"` on both the `swift` and `github-actions` ecosystems), so a bump lands as a PR into staging's branch, deploys to staging on merge, and reaches production by promotion like every other change. The configuration is read from the default branch, so the copy on main is the one that counts; security updates ignore the target and PR against main by design — a CVE fix offered straight at the production track is a feature. A new tag on a contract package arrives through the same door: dependabot PRs it against develop, and CI is the cross-repo integration test.

## Retention

Per-commit publishing fills a registry. Each repository carries a weekly scheduled workflow running the registry's delete-versions action, keeping the newest N (30 is a sane default). Consequences to know: a SHA older than the window cannot be re-pulled, so rollback targets are recent by construction, and recovery from a very old deploy is forward to a current SHA, not backward.
