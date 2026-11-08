# AGENTS.md

Shared configuration for the lsimons/ repos; see README.md.

- This repo is public. Never name or link private repos, in files, commit
  messages or PR descriptions, and keep anything that would log private repo
  data out of the public workflows.
- Gate: `mise run ci` (actionlint, Renovate config validator, zizmor).
- A change to `renovate-config.json` changes policy for every repo: say in
  the PR what it changes and why, and check it with a `dryRun=lookup` run.
