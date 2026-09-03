## Agent skills

### Issue tracker

Issues and specs are tracked with GitHub Issues via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

Triage uses the five standard Matt Pocock skill labels. See `docs/agents/triage-labels.md`.

### Domain docs

This repository uses a single-context domain documentation layout. See `docs/agents/domain.md`.


## Engineering principles

When resolving issues or implementing changes, identify and resolve relevant internally inconsistent designs and technical debt, and practice a "refactor first" approach. Operate on first principles instead of hasty patch fixes. Follow YAGNI and avoid BDUF.

For all work:

- Check assumptions.
- Don't over-engineer.
- Create a definition of done.
- Identify and resolve relevant internally inconsistent designs and technical debt; if resolution is deferred, capture it in the issue.
- Research modern best practices and implementation patterns before starting.

## Before starting work on an issue

Review the issue and confirm the plan with a human in the loop (HITL) before starting work: present the proposed plan in chat and get approval before beginning implementation.
