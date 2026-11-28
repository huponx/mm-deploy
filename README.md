# mm-deploy

ArgoCD `Application` manifests. Each environment is its own cluster with its own ArgoCD, and its root app syncs only `applications/<env>` (`destination.name: in-cluster`). Merging to `main` is the deploy: **merging to `main` is the deploy**.

```
applications/<env>/<app>.yaml   one ArgoCD Application per service and environment (dev, staging, production),
                                Helm chart from oci://ghcr.io/huponx/charts/<name>, written in the Helm repository form:
                                repoURL: ghcr.io/huponx/charts (no oci://) and chart: <name>
infra-services/<env>/  plain manifests of one environment (namespace, ExternalName Services for postgres and redis, gRPC TLS certificates), applied by the root app `infra-<env>` of mm-ansible
.github/workflows/ ci, diff, cve, pr-title, automerge, bump-revision (thin callers: the logic lives in huponx/actions)
```

## Deploy a new version

The service repo publishes a chart version to GHCR, then `spec.source.targetRevision` of its Application must point to it. Either:

- run the **Bump revision** workflow (Actions tab, or `gh workflow run bump-revision.yml -f env=dev -f app=courier -f version=1.20260813.5-main-abc1234`): it checks that the chart exists, edits the single `targetRevision` line and opens a PR labelled `automerge`;
- or edit `targetRevision` yourself in a PR (keep it a quoted string).

The PR shows a **Cluster diff** comment: the resources that change between the two revisions (Deployment, Service, ConfigMap, ...), a link to a side-by-side HTML diff, and links to the service repo compare views. With the `automerge` label the PR merges by itself once `ci` and `pr-title` pass; ArgoCD then rolls it out.

### Service repository links in the comment

The comment links `https://github.com/<repo>/compare/<old>..<new>` (two dots: the direct difference between the two commits) and `.../<old>...<new>` (three dots: the commits and changes since their common ancestor). The commit comes from the revision (`1.20260813.4-main-e1f4018` gives `e1f4018`), else from a tag `v<version>`. The repo defaults to `huponx/<application name>`; set the annotation when the name differs:

```yaml
metadata:
  annotations:
    argocd-manifest-diff/source-repo: huponx/noti
```

### Deployment PRs from the service repositories

The CI/CD of a service (for example courier, flows `main`, `staging` and `production` in `huponx/actions`) opens the PR itself with the `create-deployment-pr` action: branch `deploy/<app>/<env>/<version>`, title `chore(<app>): bump chart to <version> (<env>)`, it sets `.spec.source.targetRevision` in `applications/<env>/<app>.yaml`. Environments configured with `automerge: true` get the `automerge` label (dev); staging and production PRs wait for a review. The `ci`, `diff` and `cve` checks run on these PRs like on any other.

### db-migration (Flyway Job)

`applications/<env>/db-migration.yaml` deploys the chart of the `db-migration` repo, which runs the Flyway migrations as an ArgoCD `Sync` hook Job. All services share one database, `data` (Flyway runs as the owner role `data-user-migration`, Secret `postgres-data-migration-credentials`; services run as the DML-only role `data-user-rw`, Secret `postgres-data-rw-credentials`; both Secrets have the key `password` and are created by mm-ansible `k8s_secrets`); tables are prefixed per domain. Set the real database host per environment if it differs, and replace the sample `targetRevision` with a published version. Later versions come from the deployment PRs of the `db-migration` CI/CD.

### dictionary and dictionary-worker

`applications/<env>/dictionary.yaml` is the gRPC API; `dictionary-worker.yaml` runs the export jobs and the cleanup (an HTTP health port only, no Service, no identity settings, autoscaling by CPU). Both come from the `dictionary` repo (two apps in its `ci.yaml`, so two deployment PRs per release) and both mount the PVC `dictionary-files` (`mm-ansible`, tag `pvc`): the worker writes the parts in `exports/` and reads `images/` and `audio/`, the API streams the parts and writes the uploads. The claim is a hostpath ReadWriteOnce volume, so the two pods must run on the same node; with more nodes it needs a ReadWriteMany class or object storage. Deploy them together: without `dictionary-worker` an export job stays `queued` and nothing is cleaned. The order of a rollout does not matter (no schema change). To lower the ceiling of the export part size, set `DICTIONARY_EXPORT_PART_MAX_BYTES_CEILING` to the same value in **both** files (the API reports it to the administrators, the worker checks it when it reads the setting); the memory limit of the worker (1Gi by default) follows the ceiling.

### Callers of courier (gRPC auth)

Courier in `dev`, `staging` and `production` accepts only Kubernetes ServiceAccount tokens (no shared secret to create). `COURIER_AUTH_ALLOWED` in `applications/<env>/courier.yaml` lists the accepted callers and is empty until a caller exists: an empty list rejects every call. To let a service call courier:

1. Give the caller its own ServiceAccount (`serviceAccount.create: true` in its go-application values, or an existing name) and a projected `serviceAccountToken` volume with `audience: courier` (see the courier README for the values).
2. Add `system:serviceaccount:<namespace>:<name>` of the caller to `COURIER_AUTH_ALLOWED` (comma separated) in the courier Application of that environment, one PR per environment.

Courier's own ServiceAccount and the `system:auth-delegator` ClusterRoleBinding it needs to verify tokens come from the chart (cluster scoped, the `default` ArgoCD project allows it; a restricted AppProject must whitelist `ClusterRoleBinding`). Only `ENV=local` (a laptop) has optional auth.

### Courier as the notification service

Besides the ServiceAccount callers above, courier serves the web apps (the bell, Settings, the admin screens) through the gateway and keeps its state in PostgreSQL. `applications/<env>/courier.yaml` and `courier-worker.yaml` therefore carry:

- **PostgreSQL** (`POSTGRES_HOST/DB/USER` and the Secret `postgres-data-rw-credentials`) in both; the `courier_*` tables come from `db-migration` (V41 to V48), which must be released first.
- **identity** in `courier.yaml`: `COURIER_ISSUER` and `COURIER_JWKS_URL` (user tokens for audience `courier`), `IDENTITY_TARGET`, `IDENTITY_TOKEN_URL`, `IDENTITY_CLIENT_ID: courier` and the Secret `courier-client` (`secret`). The client `courier` (roles `service_registrar` and `user_directory`, V49) and the exchange policies `gatekeeper -> courier` and `courier -> identity` come from the `setup:` block of `identity.yaml`, which also needs the Secret `courier-client`. Courier registers its `courier:*` permissions with identity at start and exits when it cannot within two minutes, so identity's Job must have run before courier is synced.
- **Telegram and Web Push**: `TELEGRAM_BOT_USERNAME` and `VAPID_PUBLIC_KEY` in `courier.yaml`; `VAPID_SUBJECT`, `VAPID_PUBLIC_KEY` and the Secret `courier-vapid` (`private-key`) in `courier-worker.yaml`. They are empty in the files: fill them when the bot and the key pair exist (`courier/docs/setup-notification.md`); empty hides the matching switch in the portal.
- **Metrics**: both charts annotate the pods for Prometheus (`:9100`); the alert rules and the dashboard are in `courier/docs/`.
- The gateway needs no new value: the courier routes are in the gatekeeper chart and its upstream `courier` is `courier.application.svc.cluster.local:9090`. The ingress keeps the nginx default read timeout (60 s), which is above the 25 s heartbeat of the SSE stream.
- `bvchain.yaml` has a commented `BVCHAIN_NOTIFY_USER_ID` to send to the Telegram chat a user linked in the portal instead of the fixed `TELEGRAM_CHAT_ID`.

Step by step (order, checks, troubleshooting): `courier/docs/setup-notification.md` in the courier repository.

### identity and gatekeeper

`applications/<env>/identity.yaml` deploys the OAuth/OIDC provider (`identity` repo; HTTP on the ingress host, gRPC inside the cluster). Its signing key, MFA key and database password are existing Secrets (`identity-signing-key`, `identity-mfa-key`, `postgres-data-rw-credentials`), created by `mm-ansible` `k8s_secrets` from its vault (README of mm-ansible, "Identity and gatekeeper secrets"; never `kubectl create secret`); the schema comes from `db-migration`. Identity calls courier (password reset, MFA notices) with its ServiceAccount token, so `system:serviceaccount:application:identity` is in `COURIER_AUTH_ALLOWED` of every courier Application. The first admin, the `gatekeeper` client and its exchange policy come from the `setup:` block of the file (the identity setup Job, an ArgoCD PostSync hook; secrets `identity-admin` and `gatekeeper-client`). Remove that block after the Job succeeded and the admin changed the temporary password (identity README, "Initial setup Job").

`applications/<env>/bvchain.yaml` and `bvreport.yaml` deploy the two BV CronJobs of the `bvchain` repo (chart on `go-job`, values under `bv-job`; the same schedule in every environment: check-in/out `*/30 8-18 * * 1-5`, report `0 18 * * *`, `Asia/Ho_Chi_Minh`). Helm replaces lists, so each file carries the complete `env` and `secrets` lists. Secrets (in `application`, created by `mm-ansible`, tag `secrets`): `job-bv-credentials` (`bv-password`, `bv-otp-secret`) and `telegram-hung.phamqk-chat` (`chat-id`). The jobs send their Telegram messages through courier with their own ServiceAccount (`bvchain`, `bvreport`, created by the chart), so `system:serviceaccount:application:bvchain` and `...:bvreport` are in `COURIER_AUTH_ALLOWED` of every courier Application. `NOTIFICATION_SUCCESS_ENABLED` in `bvchain.yaml` turns on the message after each check-in/check-out. The `targetRevision` values are SAMPLES to replace with published versions.

`applications/<env>/gatekeeper.yaml` deploys the one shared gateway of the environment (`api.<env host>`). Web apps (`dict.`, `bill.`, ...) call it from the browser with a cookie session (CORS), mobile apps and services with `Authorization: Bearer`. A new web app is added to `GATEKEEPER_UI_ORIGINS`; no new Application for the gateway. The routes of the gateway are part of the gatekeeper chart (`routes.yaml` at the root of the `gatekeeper` repo, packaged in its chart and rendered into the ConfigMap `gatekeeper-routes`): there is nothing to apply here. A route change is a gatekeeper release, then the usual bump PR of `targetRevision`; ArgoCD applies the ConfigMap and Reloader restarts the pods. The gateway needs an exchange policy for every audience used by the routes, today `identity` (client `gatekeeper`).

`applications/<env>/{dict,bill,admin,portal}.yaml` deploy the web apps of the `web` repo (React SPA on nginx, chart `web-application`; one image for every environment). Each sets `API_BASE` (the gateway) in `application.config.runtime.data` and its ingress host; the release name must equal the app name because the chart mounts the ConfigMap `<release>-runtime-config`. For a new environment or host: add the origin to `GATEKEEPER_UI_ORIGINS` and register it as a post logout URI of the `gatekeeper` client (add it to `post_logout_uris` in the `setup:` block of `identity.yaml`, sync, then remove the block again; the Job only adds URIs). All hosts, Secret names, CIDRs and `targetRevision` values in these files are SAMPLES to replace before merging.

### TLS certificates of the gRPC servers

`infra-services/<env>/certificates/` holds one cert-manager `Certificate` per gRPC server (`identity`, `courier`, `dictionary`), issued by the `internal-ca` ClusterIssuer that mm-ansible creates. The root `infra-<env>` syncs it (recursive; the manifests set `namespace: application`). Serving TLS is switched on per hop, by `GRPC_TLS_CERT_FILE`/`GRPC_TLS_KEY_FILE` (`/var/run/tls/tls.crt`, `/var/run/tls/tls.key`) on the server and `*_INSECURE: "false"` on its clients in the same sync; the charts already mount the Secrets. Steps and rollback: `courier/docs/setup-notification.md`, section 14.

### Root app per environment

Each ArgoCD instance has a root Application whose `source.path` is `applications/<env>` of this repo. Migrating a root app that still points to `applications/` (flat files, now moved to `applications/dev/`): first make sure `applications/dev/` is merged, then switch the root app `path` to `applications/dev`, and only then delete leftovers. Switching the path first, or deleting before, makes ArgoCD prune the running applications.

## Workflows

| Workflow | Trigger | What it does |
|----------|---------|--------------|
| `ci.yml` | PR and push to `main` | `validate` (yamllint, policy checks, kubeconform for ArgoCD CRDs), `render` per changed Application (pinned chart renders with the values in git), aggregate job `ci` |
| `diff.yml` | PR | Job `diff`: the **Cluster diff** comment, the HTML page and the compare links. A thin caller of the shared action [`argocd-manifest-diff`](https://github.com/huponx/actions#argocd-manifest-diff) in `huponx/actions` |
| `cve.yml` | PR | Job `cve`: Trivy scan of the old and the new image of each changed Application (shared action [`trivy-scan`](https://github.com/huponx/actions#trivy-scan)), then the **Image vulnerabilities** comment |
| `pr-title.yml` | PR opened, edited, pushed | Conventional commit title, for example `chore(x-noti): bump chart to 1.2.3`. Job `pr-title` |
| `automerge.yml` | PR events and label changes | Label `automerge` enables squash auto-merge, removing it disables it |
| `bump-revision.yml` | manual | Opens the bump PR described above |

Required status checks for `main`: `ci`, `pr-title`, `diff` and `cve`. `diff` has no `paths:` filter so it always reports (it passes without a comment when no Application changed, and on fork PRs). It fails when the report cannot be built or the HTML pages cannot be published, which is the point of a required check: a reviewer always has the diff.

### Image vulnerabilities (`cve.yml`)

1. `plan` renders the old and the new revision of each changed Application and collects the images (action `argocd-cve-plan`). Images are paired by repository (registry and path without tag or digest). An unchanged reference is still scanned and listed (marked "Image unchanged", nothing is NEW) when its Application file is part of the PR, for example when only an env variable changed; set `CVE_INCLUDE_UNCHANGED=false` to skip it. An added one has nothing to compare with, a removed one is only mentioned.
2. `scan` runs the `trivy-scan` action once per distinct image (matrix, up to 6 in parallel) and uploads the normalised list.
3. `cve` (the required check) compares the lists (action `argocd-cve-report`) and keeps one sticky comment per PR:
   - a severity table (before, after, new, removed),
   - every vulnerability of the **new** image, sorted by severity, with the ones the old image did not have marked **NEW**,
   - a collapsed list of the vulnerabilities **removed** since the old image.

A vulnerability is identified by CVE id and package, so an upgrade that keeps the same CVE is not marked new. The check fails when planning or any scan fails (never a silent pass). By default it never fails on findings; set a gate with repository variables (**Settings > Secrets and variables > Actions > Variables**):

| Variable | Default | Effect |
|---|---|---|
| `CVE_FAIL_ON_NEW` | empty | e.g. `CRITICAL,HIGH`: fail when the PR introduces a vulnerability of these severities (including all findings of a newly added image) |
| `CVE_SEVERITY` | `UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL` | severities scanned and listed |
| `CVE_IGNORE_UNFIXED` | `false` | `true` hides vulnerabilities that have no fix yet |
| `CVE_INCLUDE_UNCHANGED` | `true` | `false` skips Applications changed without a new image |

Images are pulled with `GITHUB_TOKEN`: public images work as is, a private GHCR image needs the repository to have Read access in the package settings. Images in other private registries are not supported.

Policy checked by the action `argocd-repo-validate` (with yamllint and kubeconform): kind and apiVersion, `metadata.name` equals the file name, the ArgoCD finalizer, a pinned quoted `targetRevision` (not `latest`, `HEAD`, `main`), `destination.name`, automated sync, no secret-looking literal values (use `secrets[].secretName/key`), no `Secret` with data in `infra-services/`.

## Local checks

This repo holds no scripts: every check is a shared action in `huponx/actions` (`argocd-repo-validate`, `argocd-manifest-diff`, `trivy-scan`, `argocd-cve-plan`, `argocd-cve-report`). To run the policy check or the diff locally, clone `huponx/actions` and run the scripts under its `.github/actions/<action>/scripts/` from this repo's root (needs yq, helm, jq).

## Setup (one time)

```bash
REPO=huponx/mm-deploy
DIFFS=huponx/mm-deploy-diffs
```

1. **Merge the workflows to `main` first**, while `main` is unprotected: status checks only show up in the ruleset picker after they have run once.
2. **Actions permissions** (UI: Settings > Actions > General > Workflow permissions > Read and write):
   ```bash
   gh api -X PUT repos/$REPO/actions/permissions/workflow -f default_workflow_permissions=write
   ```
3. **Merge settings**: squash only and auto-merge enabled (`automerge.yml` needs `allow_auto_merge`).
   ```bash
   gh api -X PATCH repos/$REPO -F allow_squash_merge=true -F allow_merge_commit=false \
     -F allow_rebase_merge=false -F allow_auto_merge=true -F delete_branch_on_merge=true \
     -f squash_merge_commit_title=PR_TITLE -f squash_merge_commit_message=PR_BODY
   ```
4. **`automerge` label** (`gh pr create --label` fails if it does not exist):
   ```bash
   gh label create automerge --repo $REPO --color 0E8A16 --description "Merge automatically when checks pass"
   ```
5. **Diff pages repository** (public, GitHub Pages from a branch; it does not conflict with `huponx.github.io`: a project site is served under the same domain at `/<repo>/`):
   ```bash
   gh repo create $DIFFS --public --add-readme
   gh api -X POST repos/$DIFFS/pages -f 'source[branch]=main' -f 'source[path]=/'
   ```
   Pages are served at `https://huponx.github.io/mm-deploy-diffs/<app>/<old>...<new>.html`. To use another repo, branch or folder (for example a folder of an existing site), set repository variables; CI falls back to the values above:
   ```bash
   gh variable set DIFFS_REPO --body huponx/mm-deploy-diffs --repo $REPO
   gh variable set DIFFS_BRANCH --body main --repo $REPO
   gh variable set DIFFS_PATH --body "" --repo $REPO              # folder inside the pages repo
   gh variable set DIFFS_BASE_URL --body https://huponx.github.io/mm-deploy-diffs --repo $REPO
   ```
   > The pages are **public and not redacted**: they contain the rendered ConfigMap values, hostnames and Secret names. Keep real secrets out of `valuesObject` (reference existing Secrets). Remove a page by deleting its file from the pages repo. Pages older than `KEEP_DAYS` (default 90) are removed every Monday by the `clean` workflow of the pages repository, which also squashes its history; set the retention with `gh variable set KEEP_DAYS --body 90 --repo $DIFFS` (a variable of the pages repository, not of `mm-deploy`).
6. **`BOT_TOKEN`**: a fine-grained PAT, resource owner `huponx`, repositories **`mm-deploy` and `mm-deploy-diffs`**, permissions Contents: Read and write, Pull requests: Read and write. It is used so that PRs and pushes trigger `ci` (GITHUB_TOKEN ones do not), to merge, and to publish the diff pages.
   ```bash
   gh secret set BOT_TOKEN --repo $REPO     # prompts for the value; or: pbpaste | gh secret set BOT_TOKEN --repo $REPO
   ```
7. **Branch ruleset** for `main` (PR required, squash only, required checks `ci`, `pr-title`, `diff` and `cve`, no force push or deletion). Save as `main-ruleset.json` and apply (the file is only an input, do not commit it):
   ```json
   {
     "name": "protect main",
     "target": "branch",
     "enforcement": "active",
     "conditions": { "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] } },
     "rules": [
       { "type": "deletion" },
       { "type": "non_fast_forward" },
       { "type": "pull_request", "parameters": {
           "required_approving_review_count": 0, "dismiss_stale_reviews_on_push": false,
           "require_code_owner_review": false, "require_last_push_approval": false,
           "required_review_thread_resolution": false,
           "allowed_merge_methods": ["squash"] } },
       { "type": "required_status_checks", "parameters": {
           "strict_required_status_checks_policy": true,
           "required_status_checks": [
             { "context": "ci" }, { "context": "pr-title" }, { "context": "diff" }, { "context": "cve" } ] } }
     ],
     "bypass_actors": []
   }
   ```
   ```bash
   gh api repos/$REPO/rulesets --method POST --input main-ruleset.json
   gh api repos/$REPO/rulesets --jq '.[] | {id, name, enforcement}'   # verify
   ```
   Or in the UI: Settings > Rules > Rulesets > New ruleset > Import a ruleset. To change it later: `gh api repos/$REPO/rulesets/<id> --method PUT --input main-ruleset.json`. Rulesets on a private repo need a paid GitHub plan.
8. **Chart access for CI** (`render`, `diff`, `bump-revision` pull charts with `GITHUB_TOKEN`): charts must be Public, or each chart package must give `huponx/mm-deploy` Read access (package settings > Manage Actions access).
9. **ArgoCD**: the root app tracks `main` of this repo. Keep `main` protected, because every merge is a deployment.

## Troubleshooting

| Symptom | Cause and fix |
|---------|---------------|
| `ci` fails in `render` with "helm template failed" | The chart version in `targetRevision` does not exist, or the package is private and `mm-deploy` has no Read access (Setup step 8). |
| `check-applications.sh`: "targetRevision must be a quoted string" | Quote the version: `targetRevision: "1.20"`; unquoted numbers lose digits. |
| `check-applications.sh`: "possible secret literal" | Do not put passwords or tokens in `valuesObject`; reference a Secret with `secrets[].secretName/key`. |
| No "Cluster diff" comment | The PR is from a fork, or no Application changed (the action posts nothing then). Check the `diff` job log. |
| `diff` fails: "Unable to resolve action huponx/actions/...@v1" | `huponx/actions` has no release containing `argocd-manifest-diff` yet, or the repo does not allow access (a private `actions` repo must be shared: Settings > Actions > General > Access). |
| `cve` fails: "an image scan did not succeed" | Trivy could not pull the image (private, not found) or download its database. Open the failed `scan` job. |
| `cve` fails: "new HIGH ... vulnerability" | The gate `CVE_FAIL_ON_NEW` tripped; the comment lists the **NEW** rows. Fix by bumping to an image without them, or relax the variable. |
| `diff` fails: "could not be published" | `BOT_TOKEN` is missing, expired or has no Contents write on the pages repo; `DIFFS_BRANCH` differs from the Pages branch. |
| The HTML link gives 404 right after the comment | Pages needs up to a minute to publish; reload. If it persists, check Pages is enabled on the pages repo and `DIFFS_BASE_URL` matches its URL. |
| Compare links "unavailable" | The service repo name differs from the Application name: set `argocd-manifest-diff/source-repo`, or the revision has no commit sha and no matching tag. |
| PR does not merge by itself | Label `automerge` missing, `BOT_TOKEN` not set, auto-merge not enabled on the repo (Setup step 3), or a required check failed. |
| `Bump revision`: "was not found or is not readable" | The version is not published (check the service build) or the package is private. |
| `ci` and `pr-title` do not run on a bump PR | The PR was opened with `GITHUB_TOKEN`; the workflow uses `BOT_TOKEN` to avoid this. |

---

# Update ConfigMap CoreDNS

Managed by Ansible: set `coredns_hosts` in `mm-ansible` (`group_vars/all/coredns.yml`) and run `make deploy TAGS=coredns`.
The manual steps below are the equivalent, for reference only.

1. Edit yaml
```
kubectl -n kube-system edit configmap coredns
```

2. For postgres
```
hosts {
  192.168.0.201 postgres.my-network.local
  fallthrough
}
```

3. For redis
```
hosts {
    192.168.0.201 redis.my-network.local
    fallthrough
}
```

4. Reload ConfigMap CoreDNS
```
kubectl -n kube-system rollout restart deployment coredns
```

5. Test from Pod
```
kubectl run -it --rm busybox --image=busybox --restart=Never -- sh
nslookup postgres.my-network.local
ping postgres.my-network.local

nslookup redis.my-network.local
ping redis.my-network.local
```