# Team Fission — Postman API Tests

Git-controlled API collections, environments and test scripts for **Team Fission**.

> **Working rule:** Make changes in **Postman Local View** on a Git feature branch. Raise a pull request (PR) into `main`. After approval and merge, GitHub Actions publishes the approved files to **Fission QA** in Postman Cloud. Do not edit the shared Cloud resources directly as part of normal development.

## Quick links

| Resource | Location |
| --- | --- |
| GitHub repository | [companieshouse/fission-postman-tests](https://github.com/companieshouse/fission-postman-tests) |
| Pull requests | [Open PRs](https://github.com/companieshouse/fission-postman-tests/pulls) |
| Publishing workflow and logs | [GitHub Actions](https://github.com/companieshouse/fission-postman-tests/actions) |
| Postman workspace | [Fission QA](https://go.postman.co/workspace/Fission-QA-944f978f-1f60-4517-be5d-fadeffb731ee/overview) |
| Workspace ID | `944f978f-1f60-4517-be5d-fadeffb731ee` |
| CI/CD configuration | `.github/workflows/postman-publish.yml` |
| Local-to-Cloud mappings | `.postman/resources.yaml` |

## Contents

1. [How the solution works](#how-the-solution-works)
2. [Repository structure](#repository-structure)
3. [First-time setup for testers](#first-time-setup-for-testers)
4. [Credentials and environment variables](#credentials-and-environment-variables)
5. [Daily development workflow](#daily-development-workflow)
6. [Automated validation and Cloud publication](#automated-validation-and-cloud-publication)
7. [Permissions and ownership](#permissions-and-ownership)
8. [Maintainer procedures](#maintainer-procedures)
9. [Troubleshooting](#troubleshooting)
10. [References](#references)

## How the solution works

```text
Tester — Postman Desktop, Local View
  └── feature branch: edit, run requests, validate
        └── git commit + git push
              └── GitHub pull request → CH review/protection rules
                    ├── PR check: Postman CLI lint (no Cloud publication)
                    └── merge into main
                          └── GitHub Actions (POSTMAN_API_KEY)
                                └── postman workspace push --yes
                                      └── Fission QA — Postman Cloud
                                          (approved shared version)
```

**Source of truth:** GitHub's protected `main` branch. **Editing surface:** Local View. **Published/consumption surface:** Postman Cloud. The Postman workspace is team-scoped.

## Repository structure

```text
fission-postman-tests/
├── .github/
│   └── workflows/
│       └── postman-publish.yml             # PR lint + main-branch publication
├── .postman/
│   ├── config.json
│   └── resources.yaml                      # Workspace ID and resource mappings
├── postman/
│   ├── collections/
│   │   └── Fission QA Master.postman_collection.json
│   ├── environments/
│   │   ├── CI-DEV - TEAM Fission.environment.json
│   │   ├── CI Dev Environment (Base URL) Copy.environment.json
│   │   ├── Staging Environment (Base URL) Copy.environment.json
│   │   └── staging Copy.environment.json
│   └── globals/
│       └── workspace.postman_globals.json
└── README.md
```

The filenames above describe the initial Team Fission migration; use the actual current repository tree as the authority.

### What `.postman/resources.yaml` does

- `workspace.id` identifies **Fission QA** Workspace.
- `cloudResources.collections`, `.environments` and `.globals` map local resource paths to existing Cloud IDs.
- Correct mappings make publication update the intended Cloud objects (and avoid creating duplicates.
- When adding, renaming or deleting resources, check the mapping and review the proposed sync plan **before merging**. Resource creation can generate new IDs on the CI runner; those mappings must be reconciled back into Git.

## First-time setup for testers

### Access and prerequisites

Pre-requisites:

- GitHub account
- Postman Desktop with **Native Git / Local View** support.
- Any required CI-DEV or staging credentials obtained through approved processes (TBD).
- The separate **Postman CLI** if you want to run terminal lint/diff commands (`postman --version`). The desktop app alone does not necessarily install it.

### Clone and connect

```bash
git clone git@github.com:companieshouse/fission-postman-tests.git
cd fission-postman-tests
git switch main
git pull --ff-only origin main
```

1. Open **Postman Desktop -> Fission QA**.
2. Open **Files** and connect the cloned `fission-postman-tests` folder to the workspace (use **Open Folder** / **Connect to workspace**).
3. Select **Local View**. Confirm the selected Git branch is `main` and the collection appears under `postman/collections/`.
4. Choose the relevant environment in Postman's environment selector.
5. Set up your vault secrets as described below.

Optional CLI checks, from the repository root:

```bash
git status
postman --version
postman workspace lint
postman environment list
postman workspace context --section environment
```

If required, authenticate the local CLI using `postman login`.

## Credentials and environment variables

**Never commit real tokens, passwords, client secrets or API keys.**

### Local Vault — interactive use

Each tester should maintain their own Postman Local Vault, with the same key names referenced by the committed environments. Vault values are not stored in Git and are not shared between users.

Examples of expected variable references (the checked-in environment files determine the exact names):

```json
{
  "key": "client_secret",
  "value": "{{vault:cidev_api_key_value}}",
  "type": "secret",
  "enabled": true
}
```

In **Postman → Vault**, create the referenced keys and enter your authorised values. For example, the vault might contain `cidev_client_secret` and `cidev_api_key_value`. The **Vault key names** can be documented; **Vault values** must not be committed.

We yet need to agree how to indicate and share secret values across the team. In current version of Postman the shared vault is not available (we will probably wait for the newer version)

### What may be committed

| Type | Rule |
| --- | --- |
| Base URLs, scopes, non-sensitive configuration | Yes |
| Synthetic test-fixture IDs / data | Yes |
| Resource IDs generated during runs | Prefer generating at runtime, but generally acceptable |
| Tokens, passwords, client secrets, API keys | **Never store real values in Git**; use Vault references or approved placeholders |
| Personal data or real identity records | Do not commit |

Caveat: The Postman Local Vault is not available to the Postman CLI for CI execution. The existing GitHub Actions workflow publishes resources, but it does not run authenticated API tests. If automated test execution is added later, credentials needs to be obtained from other source, not a developer's Local Vault.

## Daily development workflow

### 1. Start from the latest `main`

```bash
git switch main
git pull --ff-only origin main
git switch -c feature/KAF-1234-description
```

### 2. Edit and test locally

- Update requests, tests, scripts or non-sensitive environment definitions in **Local View**.
- Run relevant requests with your own Local Vault credentials; review assertions and response data.
- Avoid accidental edits to other services or shared fixtures.
- Use the Postman CLI to validate file structure:

```bash
postman workspace lint
postman environment lint postman/environments
postman workspace diff --push-strategy default
```

### 3. Review and commit

```bash
git status --short
git diff --stat
git diff --check
# Review all sensitive files locally before staging.
git add postman/ .postman/   # Or stage only the specific changed files.
git diff --cached --check
git diff --cached --stat
git commit -m "ASM-1234: add API test coverage"
git push -u origin HEAD
```

### 4. Raise a pull request

1. Open [GitHub Pull Requests](https://github.com/companieshouse/fission-postman-tests/pulls) and compare the feature branch into `main`.
2. Add menaingful description: what has changed and why.
3. Confirm no sensitive values have entered files, scripts, URLs or headers.
4. Wait for reviews. Using Copilot (Balanced) review is highly recommended
5. Merge the approved PR. Do not manually push to Postman Cloud - this is done automatically.

### 5. Confirm publication and refresh your checkout

- Open [GitHub Actions](https://github.com/companieshouse/fission-postman-tests/actions) and check the new `main` run.
- Confirm authentication and **Publish to Fission QA Cloud** both succeed.
- Check the intended change in Postman **Cloud View**.
- For the next task, update your local branch from `main` with `git switch main` and `git pull --ff-only origin main`.

## Automated validation and Cloud publication

The existing workflow is **`.github/workflows/postman-publish.yml`**.

| Trigger | Behaviour |
| --- | --- |
| PR targeting `main` | Checkout -> install Postman CLI -> `postman workspace lint --fail-severity error`; Cloud authentication/publishing **skipped** |
| Push to `main` (normally merge) | Run lint -> authenticate with Postman API key -> `postman workspace push --yes` |

The secret `POSTMAN_API_KEY` is stored in **GitHub → Settings → Secrets and variables → Actions**. It authenticates the publishing identity to Postman. 

The publishing job uses the workspace bound in `.postman/resources.yaml`. 

## Permissions and ownership

| Role | Expected permissions |
| --- | --- |
| Fission Testers | Admin role |
| Others | Viewer |

## Maintaining procedures

### Investigate Git vs Cloud differences

From an up-to-date local checkout:

```bash
git switch main
git pull --ff-only origin main
postman workspace lint
postman workspace diff --push-strategy default
postman workspace context --section environment
```

**If Git is correct and Cloud is wrong:** review the differences and confirm no legitimate Cloud changes need keeping. Then a maintainer can re-run **Fission QA - Validate and Publish** in GitHub Actions with `main` selected, restoring mapped resources from approved Git. Normal/default publication does not delete unrelated Cloud-only resources.

**If Cloud has legitimate changes:** use this controlled reconciliation workflow:

1. Branch from current `main`: `git switch -c fix/reconcile-postman-cloud`.
2. In Postman **Local View -> Pull from Cloud**, select only the affected collection/environment under Update. Pull **overwrites** the corresponding local file.
3. Inspect `git diff -- postman/`, review all request moves/additions/deletions and check for secrets.
4. Keep or revise the legitimate changes; run relevant requests and `postman workspace lint`.
5. Commit, push and raise a normal reviewed PR. Merge to publish through the standard workflow.

### New or renamed environments/collections

1. Use valid Native Git filenames. For environment JSON files the suffix must be `*.environment.json`. YAML alternatives are `*.environment.yaml` and `*.environment.yml`.
2. Run `postman environment lint postman/environments` and `postman environment list`.
3. Check `.postman/resources.yaml` mappings. A mapped resource should refer to the correct existing Cloud ID. `postman workspace context --section environment` should show the local path.
4. Preview `postman workspace diff --push-strategy default` to verify *update vs create* operations.


### Manual re-publication (maintainers)

Use **GitHub -> Actions -> Fission QA - Validate and Publish -> Run workflow -> branch `main`**. Confirm its logs and the resulting Cloud state. 

## Troubleshooting

| Symptom | Check / remedy |
| --- | --- |
| `zsh: command not found: postman` | install Postman CLI and confirm `postman --version`. |
| Postman cannot see the Git folder | Check **Fission QA -> Files**, connected folder, current Git branch and Local View. |
| `.postman/resources.yaml` missing | Confirm the local repo is bound to the correct workspace; use `postman workspace connect-git WORKSPACE_ID` only if unbound. |
| `ENV009` / no matching local environments | Rename exported `*.postman_environment.json` files to `*.environment.json`, then lint and review mappings. |
| `postman workspace context` shows *cloud only* | Check local filename, discovery and `cloudResources.environments` association before publishing. |
| Environment exists in Git but not publishing log | Run `postman environment list`, lint and `postman workspace diff --push-strategy default`. Make sure the fix has been merged to `main`. |
| Collection diff shows unexpected moves/removals | Check the Postman Cloud changelog and reconcile legitimate edits via a branch/PR |
| Action fails on Postman login | Verify `POSTMAN_API_KEY` exists, has not expired. |
| Duplicate resources created | Inspect resource paths and Cloud IDs in `.postman/resources.yaml`; stop repeat publishing until mappings are corrected. |
| Request fails locally after clone | Check Local Vault key names/values. |

## References

- [Postman — Native Git overview](https://learning.postman.com/docs/use/native-git/overview/)
- [Postman — Set up Native Git and `resources.yaml`](https://learning.postman.com/docs/use/native-git/setup/)
- [Postman — Local/Cloud sync](https://learning.postman.com/docs/use/native-git/sync/)
- [Postman — Automate with CI/CD](https://learning.postman.com/docs/use/native-git/automation/)
- [Postman — CLI workspace commands](https://learning.postman.com/docs/postman-cli/postman-cli-workspace)
- [Postman — CLI environment commands](https://learning.postman.com/docs/postman-cli/postman-cli-environments)
- [Postman — Vault](https://learning.postman.com/docs/use/postman-vault/postman-vault-key)

