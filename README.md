# Images Foundry

## Problem

Kubernetes, GitLab CI, microservices, and other SDLC actors use container images, but each currently maintains its own image list. Images Foundry provides one repository containing the list of allowed images.

The `main` branch contains the list. Anyone may propose an addition or removal through a pull or merge request.

```yaml
- id: nodejs20
  description: Upstream Node.js 20 image with our updates.
```

## Image names

IDs are unique and unprefixed; builds prepend the Foundry prefix (`nodejs20` → `sdlc-nodejs20`).

## Image branches

Every merge to `main` reconciles catalog IDs with protected image branches: missing branches are created, extra branches are deleted, and matching branches are unchanged. Re-running it has the same result.

A new image gets an orphan branch named after its ID, with empty `Dockerfile` and `README.md` files. Its initial commit is authored by the `main` PR/MR author, who is also written to `CODEOWNERS`. The protected branch requires that Code Owner’s approval.

Image branches are changed only through PRs/MRs approved by their `CODEOWNERS`. Central CI runs when an image branch is created or targeted by a PR/MR, then validates and builds its Dockerfile:

```sh
docker build --check .
docker build .
```

## Image dependencies

A Foundry image may use another Foundry image in `FROM`; no manually maintained `depends_on` field is needed. During each build, CI resolves every `FROM` reference—including build arguments and multi-stage builds—to a full reference and digest. Foundry references become dependency edges; external references remain upstream provenance.

The resulting graph supports more than documentation:

- reject self-dependencies and cycles;
- find every image affected by a changed base digest;
- rebuild or mark those derived images stale;
- block removal while derived images or external consumers still use the image.

The final image also records the standard [OCI base-name and base-digest annotations](https://github.com/opencontainers/image-spec/blob/main/annotations.md). These describe its immediate base; the Foundry graph retains all build-stage dependencies.

## Consumer inventory

The inventory must answer: **which consumer uses which Foundry image, where, at which source revision, and—when resolvable—at which digest?** Reports return to a small Foundry control plane deployed from this repository, not as Git commits. The same detector supports both enforcement and tracking, but they are different controls: enforcement rejects forbidden references; reporting records allowed ones.

### Discovery

| Consumer | Observation point |
| --- | --- |
| GitLab CI | The central configuration’s inventory job reads the [merged pipeline configuration](https://docs.gitlab.com/api/lint/) and extracts job/service images; centrally owned jobs can also report their actual `CI_JOB_IMAGE`. |
| GitHub Actions | A central [reusable workflow](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows) reports the images it controls and the calling repository/revision; a repository scan covers other workflow declarations. |
| Application repositories | CI resolves images in Dockerfile `FROM` instructions and deployment/test configuration, with repository path as the location. |
| Helm repositories | CI runs [`helm template`](https://helm.sh/docs/helm/helm_template/) for every supported values set and scans the resulting Pod specifications; grepping templates is insufficient. |
| GitOps repository | CI renders the final Helm/Kustomize/Argo CD manifests and records repository path and environment. |
| Kubernetes | A [validating admission policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/) rejects forbidden images; a controller periodically snapshots images in live containers, init containers, and ephemeral containers. |

GitLab Pipeline Execution Policies can make the inventory job mandatory, but require GitLab Ultimate. GitHub reusable workflows must likewise be required by organization/repository policy. On uncontrolled runners, an arbitrary shell command can still pull an unreported image; absolute runtime enforcement requires runner, network, or registry control.

### Snapshot protocol

Each reporter submits its **complete current image set**, not individual add/delete events. A successful report replaces that consumer’s previous snapshot, so the next scan naturally captures additions, changes, and removals. An empty snapshot is valid and means that consumer no longer uses any Foundry image.

The logical Foundry ID identifies the maintained image; the resolved digest identifies the exact artifact. Store both—tags alone are mutable. A reference counts as Foundry only when its registry, namespace, and prefixed name match the configured Foundry location.

A snapshot contains:

- stable consumer ID: platform, repository or cluster, component/path, and environment;
- source revision, ordered run ID, observation time, and detector version;
- observation kind: `declared`, `live`, or `build-dependency`;
- for every occurrence: Foundry ID, requested reference, resolved digest, and file/job/workload location.

PR/MR scans validate and preview the inventory delta; only a default-branch merge replaces declared state. Scheduled scans refresh unchanged consumers and expose reporters that have stopped running. The receiver ignores older run IDs, replaces snapshots atomically, marks missed heartbeats stale, and removes a consumer only after an explicit decommission or retention timeout.

Reporters authenticate to a small ingestion API with [GitHub](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws) or [GitLab](https://docs.gitlab.com/ci/cloud_services/aws/) OIDC and may update only their own consumer ID. They do not receive permission to commit to this repository or long-lived AWS credentials.

### Storage

Use DynamoDB for current state, not Git commits:

- consumer-indexed records make a complete snapshot replaceable;
- an image-indexed GSI answers “who uses this image?”;
- a versioned snapshot is written first, then `current_generation` is switched only if its run ID is newer, preventing partial or out-of-order replacement;
- reverse-index edges carry that generation; queries verify it is current and old generations expire by TTL;
- S3 retains raw snapshots and activity history, while CI artifacts show PR/MR deltas.

Git remains the source for catalog, scanner, API, schema, and infrastructure code—not rapidly changing inventory data. Registry telemetry is a separate append-only activity source: snapshots show declared or live usage, while registry events show actual pulls. Both use the resolved image digest as their common identity.

### Lifecycle effects

- Publishing a new base digest finds and rebuilds or flags all derived Foundry images.
- A catalog-removal PR/MR lists every dependent image and consumer and is blocked while either remains.
- After consumers migrate, catalog reconciliation deletes the image branch; registry artifacts follow a separate retention policy rather than disappearing immediately.

The resulting flow is: **detect → validate against the catalog → publish a snapshot → update the dependency/consumer graph → gate unsafe image changes**.

## Primitive CI snippets

<details>
<summary>Show CI snippets</summary>

CI rejects duplicate IDs with a real shell check:

```sh
duplicates=$(sed -n 's/^[[:space:]]*- id:[[:space:]]*//p' catalog/images.yaml | sort | uniq -d)
[ -z "$duplicates" ] || { echo "duplicate IDs: $duplicates"; exit 1; }
```

The examples assume every protected non-`main` branch is an image branch. They reconcile the whole catalog on every `main` merge.

### GitHub Actions step

```yaml
- uses: actions/checkout@v7
  with:
    token: ${{ secrets.BRANCH_ADMIN_TOKEN }}

- name: Sync image branch
  env:
    GH_TOKEN: ${{ secrets.BRANCH_ADMIN_TOKEN }}
    DEFAULT_BRANCH: ${{ github.event.repository.default_branch }}
    IMAGE_OWNER: ${{ github.event.pull_request.user.login }}
    IMAGE_AUTHOR_NAME: ${{ github.event.pull_request.user.login }}
    IMAGE_AUTHOR_EMAIL: ${{ github.event.pull_request.user.login }}@users.noreply.github.com
  run: |
    desired=$(mktemp); actual=$(mktemp)
    sed -n 's/^[[:space:]]*- id:[[:space:]]*//p' catalog/images.yaml | sort -u > "$desired"
    gh api --paginate "repos/$GITHUB_REPOSITORY/branches?protected=true&per_page=100" --jq '.[].name' |
      awk -v main="$DEFAULT_BRANCH" '$0 != main' | sort -u > "$actual"

    for IMAGE_ID in $(comm -23 "$desired" "$actual"); do
      git switch --orphan "$IMAGE_ID"
      git rm -rf . 2>/dev/null || true
      mkdir -p .github
      : > Dockerfile; : > README.md
      printf '* @%s\n' "$IMAGE_OWNER" > .github/CODEOWNERS
      git add .
      git -c user.name="$IMAGE_AUTHOR_NAME" -c user.email="$IMAGE_AUTHOR_EMAIL" commit -m "Initialize $IMAGE_ID"
      git push origin "$IMAGE_ID"
      printf '%s' '{"required_status_checks":null,"enforce_admins":true,"required_pull_request_reviews":{"required_approving_review_count":1,"require_code_owner_reviews":true},"restrictions":null}' |
        gh api --method PUT "repos/$GITHUB_REPOSITORY/branches/$IMAGE_ID/protection" --input -
    done

    for IMAGE_ID in $(comm -13 "$desired" "$actual"); do
      gh api --method DELETE "repos/$GITHUB_REPOSITORY/branches/$IMAGE_ID/protection"
      git push origin --delete "$IMAGE_ID"
    done
```

### GitLab CI job

```yaml
sync-image-branch:
  stage: sync
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  script:
    - |
      git remote set-url origin "https://oauth2:${BRANCH_ADMIN_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git"
      desired=$(mktemp); actual=$(mktemp)
      sed -n 's/^[[:space:]]*- id:[[:space:]]*//p' catalog/images.yaml | sort -u > "$desired"
      curl --fail --header "PRIVATE-TOKEN: $BRANCH_ADMIN_TOKEN" \
        "$CI_API_V4_URL/projects/$CI_PROJECT_ID/protected_branches?per_page=100" |
        jq -r '.[].name' | awk -v main="$CI_DEFAULT_BRANCH" '$0 != main' | sort -u > "$actual"

      for IMAGE_ID in $(comm -23 "$desired" "$actual"); do
        git switch --orphan "$IMAGE_ID"
        git rm -rf . 2>/dev/null || true
        : > Dockerfile; : > README.md
        printf '* @%s\n' "$IMAGE_OWNER" > CODEOWNERS
        git add .
        git -c user.name="$IMAGE_AUTHOR_NAME" -c user.email="$IMAGE_AUTHOR_EMAIL" commit -m "Initialize $IMAGE_ID"
        git push origin "$IMAGE_ID"
        curl --fail --request POST --header "PRIVATE-TOKEN: $BRANCH_ADMIN_TOKEN" \
          --data-urlencode "name=$IMAGE_ID" --data "push_access_level=0" \
          --data "merge_access_level=30" --data "code_owner_approval_required=true" \
          "$CI_API_V4_URL/projects/$CI_PROJECT_ID/protected_branches"
      done

      for IMAGE_ID in $(comm -13 "$desired" "$actual"); do
        curl --fail --request DELETE --header "PRIVATE-TOKEN: $BRANCH_ADMIN_TOKEN" \
          "$CI_API_V4_URL/projects/$CI_PROJECT_ID/protected_branches/$IMAGE_ID"
        git push origin --delete "$IMAGE_ID"
      done
```

The GitLab job needs `git`, `curl`, and `jq`; `IMAGE_OWNER`, `IMAGE_AUTHOR_NAME`, and `IMAGE_AUTHOR_EMAIL` come from the merged MR metadata.

</details>

## To decide

- Catalog fields and file structure.
- Project prefix and enforcement mechanism for each SDLC actor.
