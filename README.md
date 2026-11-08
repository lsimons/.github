# lsimons/.github

Shared configuration for the [lsimons](https://github.com/lsimons) repos.
Today that is Renovate: the preset every repo extends, and the workflow that
runs it.

## Renovate

Renovate keeps dependencies current in every lsimons/ repo that commits a
`renovate.json`. It runs self-hosted in GitHub Actions as a GitHub App.

### Files

- [`renovate-config.json`](renovate-config.json): the shared preset, i.e. the
  policy.
- [`.github/workflows/renovate-run.yml`](.github/workflows/renovate-run.yml):
  one Renovate round, as a reusable workflow. All of Renovate's global config
  is here.
- [`.github/workflows/renovate.yml`](.github/workflows/renovate.yml): runs
  rounds for the public repos. Private repos have their own private caller of
  the same reusable workflow, with its own App, so their logs and Actions
  minutes stay private.
- [`renovate.json`](renovate.json): this repo opts in too, so Renovate keeps
  its own version and the workflow pins current.

### Opting a repo in

Add `renovate.json` at the repo root (and delete `.github/dependabot.yml`):

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["local>lsimons/.github:renovate-config"]
}
```

Repo-specific rules go in that file as `packageRules`, each with a
`description`. Then:

- Add the repo to the matching GitHub App installation (the public App only
  ever gets public repos; a round refuses to run otherwise).
- Turn off Dependabot security-update PRs. Keep Dependabot alerts on:
  Renovate reads them.

  ```sh
  gh api -X DELETE repos/lsimons/<repo>/automated-security-fixes
  ```

- Make sure the repo has CI that runs on pull requests. Renovate treats a PR
  without any checks as pending, so it never automerges it.

mdd-* repos are excluded for now.

### Policy (renovate-config.json)

- `config:recommended`, one PR for all non-major updates
  (`group:allNonMajor`), majors as separate PRs, weekly lock file
  maintenance. Everything automerges (rebase) once CI is green, except where
  noted below.
- Exact pins (`rangeStrategy: pin`), SHA-pinned actions with version
  comments.
- New update PRs open on Mondays (`schedule`, Europe/Amsterdam). A
  7-day rolling `minimumReleaseAge`: a release is proposed on the first
  Monday after it is a week old. Renovate only ages what it proposes;
  transitives rely on each repo's own cooldown (pnpm `minimumReleaseAge`, uv
  `exclude-newer = "1 week"`, bun, mise `minimum_release_age`), so keep those.
- Vulnerability fixes (GitHub alerts and OSV) skip both the schedule and the
  7-day wait; that is Renovate's default for `vulnerabilityAlerts`.
- TypeScript majors are disabled. Python minors (3.15 -> 3.16, across mise,
  `.python-version` and python images) get their own PR without automerge.
- Compatibility floors are never touched: `requires-python`, `engines`, the
  go.mod `go` directive (pinning would turn them into exact versions).
- Overrides (package.json `overrides`/`resolutions`, pnpm-workspace
  `overrides`) are left alone too: they are hand-maintained advisory
  floors. Vulnerability fixes still raise them.
- zizmor: the zizmor-action `version:` input is looked up on zizmor's GitHub
  releases (GHCR has no timestamps), so it ages like the mise pin and both
  land in the same PR. The action validates `version:` against its own list;
  if it doesn't know the new zizmor yet, CI goes red and nothing automerges.
- Renovate's own version (the runner's `renovate-version` input and the
  validator image in `.mise.toml`) is looked up on npm, not GHCR, for the
  same reason as zizmor: GHCR has no timestamps.
- Custom managers: `prek.toml` hook revs (tag, or SHA with a tag comment),
  and the validator image in `.mise.toml`.

osv-scanner stays in each repo's CI: it is the gate that stops automerge of
a vulnerable update, and it covers transitives that Renovate's OSV check does
not.

### How a run works

The schedule runs daily at 01:17 UTC. On Mondays that opens the week's PRs;
on other days it merges whatever is green and picks up security fixes.

Each run is a chain of rounds, so a batch goes from "due" to "merged, main
green" in one sitting:

1. Mint an App token that only reaches the repos of one visibility.
2. Run Renovate: open or update PRs, merge green automerge PRs (at most two
   per repo per round), rebase PRs that fell behind or conflict.
3. Wait for CI on Renovate's open PRs and on each default branch (up to
   about 50 minutes, the App token's lifetime).
4. Write a summary: open PRs with their checks, and default-branch CI.
5. If a green automerge PR is left, or one Renovate will rebase, dispatch
   the next round (at most 5).

A typical week takes 2 to 3 rounds. A PR that is red, or that does not
automerge, waits for you; each repo's "Dependency Dashboard" issue lists
pending, held and failed updates.

### Running it by hand

Actions > Renovate > Run workflow, or:

```sh
gh workflow run renovate.yml -R lsimons/.github                        # everything due now
gh workflow run renovate.yml -R lsimons/.github -f filter=lsimons/caseum
gh workflow run renovate.yml -R lsimons/.github -f dryRun=lookup -f force=false
```

A manual run ignores the Monday schedule unless `force=false`. Each round
uploads `renovate-report` (JSON) as an artifact:
`gh run download <id> -R lsimons/.github -n renovate-report`.

### Auth

One GitHub App per visibility, each with Administration read, Checks rw,
Commit statuses rw, Contents rw, Dependabot alerts read, Issues rw, Pull
requests rw and Workflows rw
([Renovate docs](https://docs.renovatebot.com/modules/platform/github/#running-as-a-github-app)).
The public one, `lsimons-renovate-public`, is installed on selected public
repos only. Secrets in this repo: `RENOVATE_CLIENT_ID`,
`RENOVATE_PRIVATE_KEY`.

## License

[Apache-2.0](LICENSE).
