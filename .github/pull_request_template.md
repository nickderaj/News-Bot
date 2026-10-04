## What

<!-- One paragraph: the change and why it is needed. -->

## How it was verified

<!-- Tests added, what `./scripts/ci.sh` ran, any manual check in the app. -->

## Checklist

- [ ] Title is a Conventional Commit (`feat(api): ...`); it becomes the squash commit subject
- [ ] New behaviour has tests; rules in `newsbot.domain` that decide fetching, dedup or voice-API calls are marked `# coverage-critical`
- [ ] No suppression without a `# why:` / `-- reason` justification
- [ ] No article text, clips, cookies or credentials in the diff, fixtures or this description
- [ ] Docs updated if behaviour or setup changed
