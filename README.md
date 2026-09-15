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

The first implementation is intentionally Git-only: it scans the central GitLab/GitHub CI repositories, application repositories, Helm repositories, and the GitOps repository. It does not inspect or enforce Kubernetes clusters. This covers declared state and update automation without a cluster component; it does not detect manual workloads, admission-injected sidecars, or runtime drift. A cluster collector or admission policy can be added later if that gap matters.

### Discovery

| Consumer | Observation point |
| --- | --- |
| GitLab CI | The central configuration’s inventory job reads the [merged pipeline configuration](https://docs.gitlab.com/api/lint/) and extracts job/service images; centrally owned jobs can also report their actual `CI_JOB_IMAGE`. |
| GitHub Actions | A central [reusable workflow](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows) reports the images it controls and the calling repository/revision; a repository scan covers other workflow declarations. |
| Application repositories | CI resolves images in Dockerfile `FROM` instructions and deployment/test configuration, with repository path as the location. |
| Helm repositories | CI runs [`helm template`](https://helm.sh/docs/helm/helm_template/) for every supported values set and scans the resulting Pod specifications; grepping templates is insufficient. |
| GitOps repository | CI renders the final Helm/Kustomize/Argo CD manifests and records repository path and environment. |

GitLab Pipeline Execution Policies can make the inventory job mandatory, but require GitLab Ultimate. GitHub reusable workflows must likewise be required by organization/repository policy. This rejects forbidden references committed to protected branches, but an arbitrary shell command on an uncontrolled runner can still pull an unreported image.

### Snapshot protocol

Each reporter submits its **complete current image set**, not individual add/delete events. A successful report replaces that consumer’s previous snapshot, so the next scan naturally captures additions, changes, and removals. An empty snapshot is valid and means that consumer no longer uses any Foundry image.

The Foundry ID identifies the image, its immutable release tag identifies the version, and the digest verifies the exact artifact. Store all three. A reference counts as Foundry only when its registry, namespace, and prefixed name match the configured Foundry location.

A snapshot contains:

- stable consumer ID: platform, repository, component/path, and environment;
- source revision, ordered run ID, observation time, and detector version;
- observation kind: `declared` or `build-dependency`;
- for every occurrence: Foundry ID, Foundry release version, requested reference, resolved digest, and file/job location.

PR/MR scans validate and preview the inventory delta; only a default-branch merge replaces declared state. Scheduled scans refresh unchanged consumers and expose reporters that have stopped running. The receiver ignores older run IDs, replaces snapshots atomically, marks missed heartbeats stale, and removes a consumer only after an explicit decommission or retention timeout.

Reporters authenticate to a small ingestion API with [GitHub](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws) or [GitLab](https://docs.gitlab.com/ci/cloud_services/aws/) OIDC and may update only their own consumer ID. They do not receive permission to commit to this repository or long-lived AWS credentials.

### GitLab capture flow

The central CI template adds two jobs to every tracked repository:

1. `foundry:extract` scans the checked-out commit and uploads a complete JSON snapshot. It has no AWS credentials.
2. `foundry:capture` starts a [multi-project downstream pipeline](https://docs.gitlab.com/ci/pipelines/downstream_pipelines/) in Images Foundry with `strategy: mirror`.
3. Images Foundry verifies the upstream pipeline relationship, downloads the extractor artifact from that exact pipeline, adds trusted project/revision metadata, and sends it to the ingestion API using its own OIDC identity.

The trigger and downstream capture jobs use [`resource_group`](https://docs.gitlab.com/ci/resource_groups/) keyed by consumer project and scope. This prevents overlap; it does not establish freshness. `newest_ready_first` can reduce waiting but is not a correctness mechanism. The receiver additionally accepts a snapshot only when its pipeline ID is newer and its revision is still the consumer’s default-branch head. Retries are therefore idempotent, and a late older pipeline cannot restore deleted references.

The extraction job runs on MRs for validation, but only default-branch and scheduled pipelines trigger capture. The central template must be enforced rather than merely included, or a project can remove or override it.

<details>
<summary>Show GitLab tracking PoC</summary>

Consumer `.gitlab-ci.yml`:

```yaml
include:
  - project: platform/ci-config
    ref: <immutable-commit-sha>
    file: /images-foundry/report.gitlab-ci.yml

variables:
  FOUNDRY_CONSUMER_KIND: app # ci, app, helm, or gitops
  FOUNDRY_CONSUMER_SCOPE: default
```

Central `images-foundry/report.gitlab-ci.yml`:

```yaml
foundry:extract:
  stage: .pre
  image: registry.example.com/foundry/sdlc-foundry-scanner@sha256:<digest>
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  script:
    - |
      set -eu
      found=$(mktemp)
      rendered=$(mktemp)

      find "$CI_PROJECT_DIR" -type f \( -name Dockerfile -o -name 'Dockerfile.*' \) -exec \
        awk 'FNR==1 { for (stage in stages) delete stages[stage] }
        toupper($1)=="FROM" {
          image=""; for (i=2;i<=NF;i++) if ($i !~ /^--/) { image=$i; break }
          if (!(image in stages)) print FILENAME ":" FNR "\t" image
          for (j=i+1;j<NF;j++) if (toupper($j)=="AS") stages[$(j+1)]=1
        }' {} + > "$found"

      find "$CI_PROJECT_DIR" -path '*/.git/*' -prune -o -path '*/templates/*' -prune -o \
        -type f \( -name '*.yaml' -o -name '*.yml' \) -print |
      while IFS= read -r file; do
        yq -r '.. | select(tag == "!!map" and has("image")) | .image | if tag == "!!map" then .name else . end' "$file" |
          while IFS= read -r image; do printf '%s\t%s\n' "$file" "$image"; done
        yq -r '.. | select(tag == "!!map" and has("services")) | .services[] |
          if tag == "!!map" then .name else . end' "$file" |
          while IFS= read -r image; do printf '%s\t%s\n' "$file" "$image"; done
      done >> "$found"

      find "$CI_PROJECT_DIR" -type f -name Chart.yaml -print |
      while IFS= read -r chart; do
        helm template foundry "$(dirname "$chart")" >> "$rendered"
      done
      find "$CI_PROJECT_DIR" -type f \( -name kustomization.yaml -o -name kustomization.yml \) -print |
      while IFS= read -r kustomization; do
        kustomize build "$(dirname "$kustomization")" >> "$rendered"
      done
      if [ -s "$rendered" ]; then
        yq -r '.. | select(tag == "!!map" and has("image")) | .image' "$rendered" |
          while IFS= read -r image; do printf '%s\t%s\n' rendered "$image"; done >> "$found"
      fi

      sort -u "$found" | awk -F '\t' '$2 != "scratch"' > foundry-images.tsv
      invalid=$(cut -f2 foundry-images.tsv | grep -Ev '^registry\.example\.com/foundry/sdlc-[a-z0-9._-]+([:@][A-Za-z0-9._:+-]+)?$' || true)
      [ -z "$invalid" ] || { printf 'Forbidden or unresolved images:\n%s\n' "$invalid"; exit 1; }

      jq -Rn --arg kind "$FOUNDRY_CONSUMER_KIND" --arg scope "$FOUNDRY_CONSUMER_SCOPE" \
        '[inputs | split("\t") | {location: .[0], requested_ref: .[1]}] |
         {kind: $kind, scope: $scope, images: .}' \
        < foundry-images.tsv > foundry-snapshot.json
  artifacts:
    paths: [foundry-snapshot.json]
    expire_in: 1 hour

foundry:capture:
  stage: .post
  needs: [foundry:extract]
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  resource_group: "foundry-trigger-$CI_PROJECT_ID-$FOUNDRY_CONSUMER_SCOPE"
  variables:
    UPSTREAM_PROJECT_ID: $CI_PROJECT_ID
    UPSTREAM_PIPELINE_ID: $CI_PIPELINE_ID
    UPSTREAM_COMMIT_SHA: $CI_COMMIT_SHA
    UPSTREAM_REF: $CI_COMMIT_REF_NAME
    UPSTREAM_SCOPE: $FOUNDRY_CONSUMER_SCOPE
  trigger:
    project: platform/images-foundry
    branch: main
    strategy: mirror
    forward:
      yaml_variables: true
      pipeline_variables: false
```

Images Foundry `.gitlab-ci.yml` capture fragment:

```yaml
capture-consumer:
  stage: capture
  image: registry.example.com/foundry/sdlc-foundry-capture@sha256:<digest>
  rules:
    - if: $CI_PIPELINE_SOURCE == "pipeline"
  resource_group: "inventory-$UPSTREAM_PROJECT_ID-$UPSTREAM_SCOPE"
  id_tokens:
    FOUNDRY_OIDC_TOKEN:
      aud: https://images-foundry.example.com
  script:
    - |
      set -eu
      echo "$UPSTREAM_PROJECT_ID:$UPSTREAM_PIPELINE_ID:$UPSTREAM_SCOPE" | grep -Eq '^[0-9]+:[0-9]+:[a-z0-9._-]+$'

      pipeline=$(curl --fail --header "PRIVATE-TOKEN: $GITLAB_READ_TOKEN" \
        "$CI_API_V4_URL/projects/$UPSTREAM_PROJECT_ID/pipelines/$UPSTREAM_PIPELINE_ID")
      printf '%s' "$pipeline" | jq -e --arg sha "$UPSTREAM_COMMIT_SHA" --arg ref "$UPSTREAM_REF" \
        '.sha == $sha and .ref == $ref' >/dev/null

      bridges=$(curl --fail --header "PRIVATE-TOKEN: $GITLAB_READ_TOKEN" \
        "$CI_API_V4_URL/projects/$UPSTREAM_PROJECT_ID/pipelines/$UPSTREAM_PIPELINE_ID/bridges")
      printf '%s' "$bridges" | jq -e --argjson downstream "$CI_PIPELINE_ID" \
        'any(.[]; .downstream_pipeline.id == $downstream)' >/dev/null

      jobs=$(curl --fail --header "PRIVATE-TOKEN: $GITLAB_READ_TOKEN" \
        "$CI_API_V4_URL/projects/$UPSTREAM_PROJECT_ID/pipelines/$UPSTREAM_PIPELINE_ID/jobs?scope=success")
      extract_job=$(printf '%s' "$jobs" | jq -er '[.[] | select(.name == "foundry:extract")][0].id')
      curl --fail --header "PRIVATE-TOKEN: $GITLAB_READ_TOKEN" \
        "$CI_API_V4_URL/projects/$UPSTREAM_PROJECT_ID/jobs/$extract_job/artifacts/foundry-snapshot.json" \
        --output extracted.json

      jq -e '(.images | type == "array") and (.images | length <= 10000)' extracted.json >/dev/null
      jq --arg consumer "gitlab:$UPSTREAM_PROJECT_ID:$UPSTREAM_SCOPE" \
         --arg revision "$UPSTREAM_COMMIT_SHA" --argjson run_id "$UPSTREAM_PIPELINE_ID" \
         --arg observed_at "$(date -u +%FT%TZ)" \
        '. + {consumer: $consumer, revision: $revision, run_id: $run_id,
              observed_at: $observed_at, observation: "declared"}' \
        extracted.json > snapshot.json

      curl --fail --request PUT \
        --header "Authorization: Bearer $FOUNDRY_OIDC_TOKEN" \
        --header 'Content-Type: application/json' \
        --data-binary @snapshot.json \
        "$FOUNDRY_INGEST_URL/v1/snapshots"
```

`GITLAB_READ_TOKEN` is a masked, protected, read-only group token available only in Images Foundry. The ingestion service revalidates the schema, Foundry registry/name, source head, and monotonically increasing run ID before switching `current_generation`. The shell extractor is intentionally a literal-reference PoC; production extractors must resolve CI variables and render every supported Helm/GitOps values set.

</details>

### Storage

Use DynamoDB for current state, not Git commits:

- consumer-indexed records make a complete snapshot replaceable;
- an image-indexed GSI answers “who uses this image?”;
- a versioned snapshot is written first, then `current_generation` is switched only if its run ID is newer, preventing partial or out-of-order replacement;
- reverse-index edges carry that generation; queries verify it is current and old generations expire by TTL;
- CI artifacts show PR/MR inventory deltas.

Git remains the source for catalog, scanner, API, schema, and infrastructure code—not rapidly changing inventory data. Actual registry pulls are outside the initial Git-only scope.

### Lifecycle effects

- Publishing a new base digest finds and rebuilds or flags all derived Foundry images.
- A catalog-removal PR/MR lists every dependent image and consumer and is blocked while either remains.
- After consumers migrate, catalog reconciliation deletes the image branch; registry artifacts follow a separate retention policy rather than disappearing immediately.

The resulting flow is: **detect → validate against the catalog → publish a snapshot → update the dependency/consumer graph → gate unsafe image changes**.

## Automated patching and releases

Even a verbatim upstream copy is valuable: from its first release it has a Foundry name, owner, build pipeline, scan history, and controlled publication path. At first it may share every filesystem layer with upstream; later Foundry can publish patched releases without waiting for upstream.

### Daily patch loop

The scheduled pipeline on `main` enumerates the catalog and starts one patch pipeline on each image branch. Patching is serialized with `resource_group: patch-$IMAGE_ID`; a second conditional check confirms that the source digest is still current before publication.

Use different tools for different changes:

- Renovate updates an upstream tag or digest in `FROM`.
- Trivy produces the vulnerability report; [Copacetic](https://project-copacetic.github.io/copacetic/website/) creates scanner-driven OS-package patch layers from that report.
- A free-form AI agent may propose an ordinary MR, but its changes are never eligible for approval-free merge.

The first automatic scope is fixable OS packages. Application-library vulnerabilities require a source/upstream update and normal review unless a separate deterministic updater is introduced. Ad-hoc generated `RUN apt install ...` lines are not the interface: they are distro-specific and accumulate in Dockerfiles; the durable input is a patch lock, and the builder chooses the appropriate package mechanism.

The automatic patch bot may change only the upstream digest and a machine-owned patch lock containing the source digest, scanner database version, affected packages, and fixed versions. When it finds a fix, it creates or updates `foundry-patch/$IMAGE_ID` targeting the image branch. The branch pipeline builds the candidate, runs image tests, rescans it, generates an SBOM and provenance, and rejects regressions or remaining fixable vulnerabilities above the configured severity. A free-form Dockerfile change still requires its Code Owner.

If no fix is available or the resulting digest is unchanged, the run records the scan and creates no MR.

<details>
<summary>Show GitLab patch-candidate job</summary>

`CURRENT_IMAGE` is the published digest reference. The MR pipeline builds and scans a temporary candidate; publication happens only after merge.

```yaml
patch-candidate:
  stage: test
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME =~ /^foundry-patch\//'
  resource_group: "patch-$CI_MERGE_REQUEST_TARGET_BRANCH_NAME"
  script:
    - trivy image --vuln-type os --ignore-unfixed --format json --output before.json "$CURRENT_IMAGE"
    - PATCH_TAG="candidate-$CI_PIPELINE_ID"
    - copa patch --image "$CURRENT_IMAGE" --report before.json --tag "$PATCH_TAG"
    - CANDIDATE_IMAGE="${CURRENT_IMAGE%@*}:$PATCH_TAG"
    - trivy image --vuln-type os --ignore-unfixed --severity HIGH,CRITICAL --exit-code 1 "$CANDIDATE_IMAGE"
    - docker push "$CANDIDATE_IMAGE"
```

</details>

### Approval exception

Do not make the patch author a general Code Owner. Use a separate, CI-only merge service account that can bypass Code Owner approval for patch MRs. Before enabling [auto-merge](https://docs.gitlab.com/user/project/merge_requests/auto_merge/), it verifies:

- the MR author, source-branch prefix, target image branch, and allowed file paths;
- the patch lock refers to the currently published digest;
- the required build, test, rescan, SBOM, and provenance jobs passed for the MR head;
- the candidate digest is exactly the digest recorded by those jobs.

GitLab auto-merge does not itself bypass missing Code Owner approval. Native branch protection therefore has an unavoidable trade-off: granting the merge service account [`Allowed to push and merge`](https://docs.gitlab.com/user/project/repository/branches/protected/) bypasses Code Owner approval but also permits that account to push directly. The credential must exist only in a protected finalizer job that performs the checks above; everyone else remains denied direct push. GitLab Ultimate approval-policy service-account exceptions can make this bypass explicit and auditable, but do not replace testing the effective branch rules.

Human and unrestricted bot changes retain the normal Code Owner requirement. The image project also requires a successful, non-skipped MR pipeline, but that setting alone cannot constrain a credential which is allowed to push directly.

### Publication and versions

Every accepted build receives an immutable Foundry release tag and digest, for example `sdlc-nodejs20:20.18.1-foundry.3@sha256:…`. The tested candidate digest is promoted after merge instead of rebuilding different bytes. A mutable convenience tag such as `current` may exist, but consumers must use an immutable release or digest; otherwise Git cannot show or review an update.

Each release records image ID, Foundry version, digest, upstream/base digest, image-branch commit, patch input, scan before and after, SBOM, provenance, and publication time.

### Consumer rollout

Publication starts a rollout job in Images Foundry:

1. Query the image-indexed consumer inventory for consumers on older releases or digests.
2. Group occurrences by repository and create one MR per repository that updates every occurrence of that image.
3. Reuse a stable branch such as `foundry-update/$IMAGE_ID`; if a newer release appears, update the open MR instead of creating another.
4. Request auto-merge, but let the consumer repository’s required pipeline and merge policy decide whether it succeeds.
5. After merge, the consumer’s normal default-branch capture replaces its snapshot with the new version and digest.

Renovate can implement the update MR when it understands the Foundry tag scheme; the custom coordinator still uses the inventory to target the exact repositories and report rollout status.

Rollout jobs use `resource_group: rollout-$CONSUMER_PROJECT_ID-$IMAGE_ID`. The expected old digest is checked before editing, so concurrent human changes are not overwritten. Base-image releases first rebuild dependent Foundry images in graph order; consumers are updated only to the resulting derived releases.

The inventory can then show `current`, `update pending`, `pipeline failed`, `stale report`, or `unreported`. Consumers remaining on the old digest are visible immediately, with links to their update MR and latest pipeline.

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
