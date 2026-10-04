# Working in News-Bot

Read [README.md](README.md) for the product overview,
[docs/PLAN.md](docs/PLAN.md) for the build plan and
[docs/DEVELOPMENT.md](docs/DEVELOPMENT.md) for the gate every change must pass.

- Never push to `main`. Branch, open a PR with a Conventional Commit title, and
  let the owner review it.
- Run `./scripts/ci.sh` before pushing; do not open a PR that fails it.
- Do not weaken the gate (thresholds, rule selections, required checks,
  `scripts/policy_check.py`) inside a feature PR. Gate changes get their own
  `ci:` PR explaining why.
- `newsbot.domain` is pure: no I/O, no framework imports. Rules that decide
  what gets fetched, deduplicated, or sent to the paid voice API are marked
  `# coverage-critical`.
- Tests are hermetic: no network, no real FT/Bloomberg login, no real xAI or
  model calls, no Tailscale, no production database. Use recorded HTML
  fixtures written for the tests, never saved copies of real articles.
- Never call the xAI voice API outside the explicit "listen" action, and never
  for an article that already has a stored clip.
- Pin every new dependency exactly and justify it in the PR description.
- **The repository is public.** Never commit hostnames, tailnet names, IPs,
  emails, host paths, service accounts, ports in use, credentials, cookies,
  browser profiles, or personal data. **Never commit article text, headlines
  from the subscriptions, or voice clips**: they are licensed content.
  Host-specific values go in the git-ignored `deploy/local.env`; articles,
  clips and sessions stay in the private data directory on the host.
