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
