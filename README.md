# open-macaron/config

Who [macaron-bot](https://github.com/open-macaron/macaron) works for, and how: the trusted projects, their repositories, the review bots whose comments count, budgets, caps, models and environments. Who may trigger it is not here: it's an account of the macaron web app, invited by an administrator, with the capabilities they granted. The controller reads it from this repository's [published snapshot](#published-snapshot), and a merged change is live within seconds, without a deploy.

Everything here is public. The file formats are JSON Schemas in [`open-macaron/macaron`](https://github.com/open-macaron/macaron/tree/main/packages/contracts/schemas) (`config-*.schema.json`), and its [SCALING.md §2](https://github.com/open-macaron/macaron/blob/main/docs/SCALING.md) explains the hierarchy.

## Layout

```
global.yaml                          defaults for every repository, global caps (platform team)
projects/<project>/
├── OWNERS.yaml                      who approves changes in this folder
├── project.yaml                     review bots, budget, concurrency, project defaults
└── repos/<repo>.yaml                one upstream repository: its fork, commands, overrides
```

A repository's settings are `global.yaml` → its `project.yaml` → its `repos/<repo>.yaml`, each level overriding the previous one.

## Owners and approvals

Each project folder has an `OWNERS.yaml`:

```yaml
owners:
  - login: axel7083     # shown and @mentioned
    id: 42176370        # what approvals are matched on
```

Find an account's id with `gh api users/<login> --jq .id`. **The id is what counts**: a login can be renamed and then registered by someone else. The check fails when a login no longer belongs to its id, so the file gets updated, but access never moves to whoever holds the old name.

**Owners don't need write access to this repository.** GitHub doesn't read `OWNERS.yaml`; our workflows do:

1. Anyone opens a pull request, from a fork (nobody pushes branches here).
2. `check` validates the files exactly as the controller does and annotates problems. It is feedback only.
3. An owner of each project folder the pull request touches reviews it on GitHub and approves.
4. Each review wakes up `merge`, which decides from scratch and merges when it can. It keeps one comment on the pull request saying what is still missing, and @mentions the owners it waits for.

`merge` merges a pull request only when **all** of these hold:

- every changed path (both sides of a rename) is inside `projects/<project>/`, and that folder's `OWNERS.yaml` **already exists on `main`**;
- for each of those folders, an owner listed in its `OWNERS.yaml` **on `main`** (by id) has a standing approval on **the latest commit** of the pull request, and no owner of that folder has a standing "changes requested";
- the author's own approval doesn't count;
- **no file changes a setting reserved for the platform team**: `users` (deprecated, ignored), `bots`, `budget`, `concurrency`, `budgets`, `caps`, `reviewCost`, `runsOn` or `platforms`, at the top level or under a project's `defaults`. These decide whose comments count and what the bot may spend or run on;
- the files are valid, checked by the pinned `config-cli` with the controller's own validation.

It then merges exactly the commit it checked (squash): if something was pushed in the meantime, GitHub refuses.

Consequences:

- **Changing the content after an approval** needs a new approval: `main`'s ruleset dismisses approvals when a push changes what the pull request proposes.
- **A pull request touching two projects** needs an owner of each.
- **Owners manage their own `OWNERS.yaml`:** a change to it is approved by the owners currently on `main`.
- **Owners onboard repositories on their own:** a new `repos/<repo>.yaml` that sets no reserved setting is merged on their approval.
- **The platform team merges by hand** changes to reserved settings, `global.yaml`, new project folders, `.github/`, this README and other root files.

### Why it's safe

Approvals are checked by a workflow rather than GitHub's branch protection, so the workflows assume the worst about pull requests:

- For a pull request from a fork, GitHub runs **the pull request's own copy** of `check.yml` and `review.yml`. They get no write permission and no secret, and nothing trusts what they do: they only wake `merge` up.
- `merge.yml` always runs **from `main`**. From the event that woke it up it uses only the commit SHA, to find which open pull request to look at, never as proof of anything.
- Everything `merge` decides on comes from GitHub's API or from `main`: reviews, changed files, `OWNERS.yaml`. The pull request's files are only read as data by `config-cli`, which is downloaded from a pinned release and checked by sha256. They can only make a merge stricter (reserved settings), never allow one. Nothing from the pull request is executed.
- Only the job that merges has write permission. It never checks out the pull request. The job that does check it out (to validate it) can only read, and no workflow here uses a cache.
- The repository has no secrets.

On top of this, the controller refuses a snapshot it can't validate or whose models the LLM backend can't price, keeping the previous one.

## Published snapshot

On every merge, `pages` publishes the configuration on GitHub Pages:

```
https://open-macaron.github.io/config/v1/config.json     { version, commit, files: { "<path>": "<yaml>" } }
```

The files verbatim, at one commit (schema `config-published.schema.json`). When the Pages deployment succeeds, GitHub notifies the controller (`deployment_status` webhook), which fetches that commit, validates it as a local tree, and applies it. It also keeps the last applied snapshot, to start from when Pages is unavailable.

## Checking locally

```sh
# The release pinned in .github/workflows (CONFIG_CLI_URL):
curl -fsSLo config-cli.mjs 'https://github.com/open-macaron/macaron/releases/download/%40open-macaron%2Fconfig-cli%40<version>/config-cli.mjs'
node config-cli.mjs validate .                       # the controller's checks, OWNERS.yaml
GITHUB_TOKEN=$(gh auth token) node config-cli.mjs validate . --check-logins
```

`config-cli` lives in [`open-macaron/macaron`](https://github.com/open-macaron/macaron/tree/main/packages/config-cli). Each release has a build provenance attestation: `gh attestation verify config-cli.mjs -R open-macaron/macaron`.
