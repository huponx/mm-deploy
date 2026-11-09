# mm-deploy

ArgoCD `Application` manifests. Each environment is its own cluster with its own ArgoCD, and its root app syncs only `applications/<env>` (`destination.name: in-cluster`). Merging to `main` is the deploy: **merging to `main` is the deploy**.

```
applications/<env>/<app>.yaml   one ArgoCD Application per service and environment (dev, staging, production),
                                Helm chart from oci://ghcr.io/huponx/charts/<name>
infra-services/    plain manifests (namespace, ExternalName Services for postgres and redis)
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

1. `plan` renders the old and the new revision of each changed Application and collects the images (action `argocd-cve-plan`). Images are paired by repository (registry and path without tag or digest). An unchanged reference is skipped, an added one has nothing to compare with, a removed one is only mentioned.
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
   > The pages are **public and not redacted**: they contain the rendered ConfigMap values, hostnames and Secret names. Keep real secrets out of `valuesObject` (reference existing Secrets). Remove a page by deleting its file from the pages repo. Pages older than 90 days are removed automatically on the next publish.
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