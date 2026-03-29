# Setting up Dependabot for an Nx Angular project

This guide explains how to configure Dependabot for a repository that uses an
**Nx monorepo** with **Angular** applications, and how to keep the
auto-generated PR branch names short and clean.

---

## Prerequisites

| Tool | Minimum version |
|------|-----------------|
| Node.js | 18 LTS |
| Nx CLI | 17+ |
| Angular | 17+ |
| GitHub repository | any visibility |

---

## 1 – Repository layout

Nx stores a single root `package.json` (and the corresponding lock file) at
the repo root.  Every application and library inside the `apps/` and `libs/`
folders shares those dependencies.

```
my-nx-workspace/
├── .github/
│   ├── dependabot.yml          ← Dependabot configuration
│   └── workflows/
│       ├── ci.yml
│       └── rename-dependabot-branch.yml   ← short branch names (see §4)
├── apps/
│   └── my-app/
├── libs/
├── nx.json
├── package.json                ← single lock-file here
└── package-lock.json
```

---

## 2 – `dependabot.yml` configuration

Create (or update) `.github/dependabot.yml`.

Because Nx uses a **single root `package.json`**, set `directory: "/"` for the
`npm` entry.  This is the key difference from non-Nx projects where different
apps may live in sub-directories.

```yaml
version: 2
updates:
  # ── npm / yarn ──────────────────────────────────────────────────────────
  # Nx keeps one package.json at the repo root.
  - package-ecosystem: "npm"
    directory: "/"               # ← always "/" for Nx monorepos
    schedule:
      interval: "daily"          # or "weekly" if you prefer less noise
    # Using "-" as separator gives Docker-tag-safe branch names and removes
    # the extra "/" characters that would otherwise appear.
    pull-request-branch-name:
      separator: "-"
    # Optional: only open PRs for these labels
    # labels:
    #   - "dependencies"
    #   - "angular"
    # Optional: auto-merge patch updates
    # open-pull-requests-limit: 10

  # ── GitHub Actions ───────────────────────────────────────────────────────
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    pull-request-branch-name:
      separator: "-"

  # ── Docker (if you have a Dockerfile) ───────────────────────────────────
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    pull-request-branch-name:
      separator: "-"
```

### Why `directory: "/"`?

Dependabot appends the `directory` value to the branch name when it is not
`/`.  Without this setting your branches look like:

```
dependabot/npm_and_yarn/javascript/lodash-4.18.0   ← with directory "/javascript"
dependabot-npm_and_yarn-lodash-4.18.0              ← with directory "/"  ✓
```

---

## 3 – Branch name anatomy

Even with `directory: "/"`, Dependabot still prepends its own prefix:

```
dependabot-npm_and_yarn-<package>-<version>
│           │            └─ what you actually care about
│           └─ ecosystem (always present, not configurable)
└─ fixed prefix (not configurable)
```

To get a short, readable branch name like `bot/<package>-<version>` you need
the rename workflow described in §4.

---

## 4 – Automatic branch renaming (bot/\<package\>-\<version\>)

Dependabot does not allow a custom branch prefix through configuration alone.
The rename workflow below fires every time Dependabot opens a PR and renames
the branch from:

```
dependabot-npm_and_yarn-hot-formula-parser-4.0.0
```

to:

```
bot/hot-formula-parser-4.0.0   (≤ 45 characters)
```

### How it works

1. The `pull_request: opened` trigger fires only when `github.actor == 'dependabot[bot]'`.
2. The shell script strips the `dependabot-` and `<ecosystem>-` prefixes from the branch name.
3. It calls the GitHub Branches API (`POST /repos/{owner}/{repo}/branches/{branch}/rename`) to rename the branch.
4. GitHub automatically updates the PR's `head` ref so the PR stays linked.

Save the workflow to `.github/workflows/rename-dependabot-branch.yml`:

```yaml
name: Rename Dependabot Branch

on:
  pull_request:
    types: [opened]

jobs:
  rename-branch:
    if: github.actor == 'dependabot[bot]'
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - name: Rename to bot/<package-version> (max 45 chars)
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          BRANCH: ${{ github.head_ref }}
          REPO: ${{ github.repository }}
        run: |
          REMAINDER="${BRANCH#*-}"       # strip "dependabot-"
          REMAINDER="${REMAINDER#*-}"    # strip "<ecosystem>-"
          NEW="bot/${REMAINDER}"
          NEW="${NEW:0:45}"
          ENCODED=$(python3 -c \
            "import urllib.parse, sys; print(urllib.parse.quote(sys.argv[1], safe=''))" \
            "$BRANCH")
          gh api \
            --method POST \
            -H "Accept: application/vnd.github+json" \
            "/repos/${REPO}/branches/${ENCODED}/rename" \
            -f new_name="${NEW}"
```

> **Permissions note** – The workflow uses `secrets.GITHUB_TOKEN` which is
> automatically available.  No extra secrets are required.  The `contents:
> write` permission is needed to rename the branch; `pull-requests: write` is
> needed to keep the PR head ref in sync.

---

## 5 – Angular-specific tips

### Ignore major Angular version bumps

Angular requires coordinated upgrades (CLI, core, CDK, Material, …).
Use `ignore` rules so Dependabot only proposes patch/minor updates for Angular
packages and you handle major upgrades yourself via `nx migrate`.

```yaml
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    pull-request-branch-name:
      separator: "-"
    ignore:
      # Let nx migrate handle Angular major versions.
      - dependency-name: "@angular/*"
        update-types: ["version-update:semver-major"]
      - dependency-name: "@angular-devkit/*"
        update-types: ["version-update:semver-major"]
      - dependency-name: "@nrwl/*"
        update-types: ["version-update:semver-major"]
      - dependency-name: "@nx/*"
        update-types: ["version-update:semver-major"]
```

### Group related updates

Reduce PR noise by grouping Angular or Nx packages into a single PR:

```yaml
    groups:
      angular:
        patterns:
          - "@angular/*"
          - "@angular-devkit/*"
      nx:
        patterns:
          - "@nrwl/*"
          - "@nx/*"
```

---

## 6 – Enabling Dependabot security updates

Security alerts and automated security PRs are separate from version updates.
Enable them in **Settings → Code security and analysis**:

- ✅ Dependency graph
- ✅ Dependabot alerts
- ✅ Dependabot security updates

No changes to `dependabot.yml` are needed for security updates.

---

## 7 – Verifying the setup

After merging `dependabot.yml`:

1. Go to **Insights → Dependency graph → Dependabot** in your repository.
2. You should see entries for `npm`, `github-actions`, and (optionally) `docker`.
3. Click **"Last checked …"** next to `npm` to trigger a manual check.
4. The next Dependabot PR should have a branch named `bot/<package>-<version>`.

---

## 8 – Full example (`dependabot.yml`)

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "daily"
    pull-request-branch-name:
      separator: "-"
    open-pull-requests-limit: 10
    labels:
      - "dependencies"
    ignore:
      - dependency-name: "@angular/*"
        update-types: ["version-update:semver-major"]
      - dependency-name: "@nrwl/*"
        update-types: ["version-update:semver-major"]
      - dependency-name: "@nx/*"
        update-types: ["version-update:semver-major"]
    groups:
      angular:
        patterns: ["@angular/*", "@angular-devkit/*"]
      nx:
        patterns: ["@nrwl/*", "@nx/*"]

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    pull-request-branch-name:
      separator: "-"

  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    pull-request-branch-name:
      separator: "-"
```
