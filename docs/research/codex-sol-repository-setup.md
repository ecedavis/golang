# Setting up a repository for Codex with GPT-5.6 Sol and Matt Pocock's skills

Research date: 2026-08-27

## Recommendation

Use four separate layers, each for a different scope:

1. Put personal model and reasoning defaults in the user-level Codex config, `~/.codex/config.toml` (`%USERPROFILE%\.codex\config.toml` on Windows).
2. Put durable repository guidance and exact setup/test/build commands in a root `AGENTS.md`, with nested instruction files only where a subtree genuinely differs.
3. Keep environment bootstrapping in repeatable repository-owned commands, then have Codex local/cloud setup call those commands.
4. Keep Matt Pocock's skills installed globally, but explicitly run `$setup-matt-pocock-skills` once in every repository and commit the Markdown configuration it produces.

This separation matters because Codex treats user config, trusted project config, `AGENTS.md`, environment setup, and skills as different configuration surfaces. User defaults live in `~/.codex/config.toml`; a trusted repository can override them with `.codex/config.toml`. [OpenAI: Config basics](https://developers.openai.com/codex/config-file/config-basic)

## 1. Configure GPT-5.6 Sol at the correct scope

For a personal default across repositories, use:

```toml
# ~/.codex/config.toml
model = "gpt-5.6-sol"
model_reasoning_effort = "high"
```

OpenAI identifies `gpt-5.6-sol` as the flagship-capability variant; the shorter `gpt-5.6` alias currently routes to Sol. OpenAI recommends `medium` as the balanced starting point, `high` or `xhigh` only where deeper reasoning produces a measured gain, and `max` for the hardest quality-first workloads. [OpenAI: GPT-5.6 model guidance](https://developers.openai.com/api/docs/guides/latest-model)

`high` is a pragmatic default for substantial repository work. Use `medium` when latency matters, and evaluate `xhigh` on representative hard tasks rather than assuming more reasoning is always better. There is a current documentation mismatch worth knowing: the GPT-5.6 model guide lists effort through `max`, while the Codex configuration reference currently enumerates `minimal`, `low`, `medium`, `high`, and `xhigh` for `model_reasoning_effort`. Until the Codex reference catches up, `high` or `xhigh` is the portable checked-in/default value; use `max` only if the installed Codex client explicitly offers or accepts it. [OpenAI: GPT-5.6 model guidance](https://developers.openai.com/api/docs/guides/latest-model) [OpenAI: Codex configuration reference](https://developers.openai.com/codex/config-file/config-reference)

If the whole team intentionally wants this repository to override each developer's default, the same keys can instead be committed in:

```toml
# .codex/config.toml
model = "gpt-5.6-sol"
model_reasoning_effort = "high"
```

Codex resolves CLI overrides first, then project `.codex/config.toml` files from repository root toward the current directory, then a selected profile, then user config. Project config is loaded only for trusted projects. Therefore, keep personal cost/latency preferences global; commit model/effort only when they are a real project contract. [OpenAI: configuration precedence](https://developers.openai.com/codex/config-file/config-basic#configuration-precedence)

Do not put model selection in `AGENTS.md`. `AGENTS.md` is behavioral/project guidance, while `config.toml` is the actual Codex setting surface.

## 2. Make `AGENTS.md` the repository entry point

Codex builds an instruction chain once per run. It reads global `~/.codex/AGENTS.override.md` or `~/.codex/AGENTS.md`, then walks from the repository root to the current working directory. In each directory it selects at most one of `AGENTS.override.md`, `AGENTS.md`, or a configured fallback filename. Files closer to the working directory appear later and therefore override broader guidance. The combined project instruction budget is 32 KiB by default. [OpenAI: AGENTS.md discovery](https://developers.openai.com/codex/agent-configuration/agents-md#how-codex-discovers-guidance)

The root `AGENTS.md` should be concise but operational. Include:

- The project's purpose and a short architecture map.
- The supported runtime/toolchain versions and the one canonical setup command.
- Exact commands for a targeted test, full tests, formatting, lint/static analysis, type checking where relevant, building, and a minimal smoke test.
- The directory each command must run from, plus any prerequisites.
- Which generated files must not be hand-edited and how to regenerate them.
- Migration, dependency, secret, external-write, and destructive-action boundaries.
- The repository's definition of done: which validation is required for which type of change, and what to report if a check cannot run.
- Pointers to domain documentation and the Matt skill configuration under `docs/agents/`.

OpenAI's model guidance recommends outcome-first instructions, clear success criteria and authorization boundaries, and concrete validation such as targeted tests, type/lint checks, affected-package builds, and a minimal smoke test. It also recommends keeping prompts lean and stating each instruction once. [OpenAI: GPT-5.6 prompting guidance](https://developers.openai.com/api/docs/guides/latest-model)

A practical starting shape is:

```markdown
# AGENTS.md

## Project

- Purpose: <one paragraph>
- Architecture: <key modules and seams>
- Domain vocabulary: see `CONTEXT.md`.
- Decisions: see `docs/adr/`.

## Setup

- Required toolchain: <versions or version-manager files>
- Bootstrap from repo root: `<exact command>`
- Required local services: <commands or "none">
- Environment variables: copy `<example file>`; never commit secrets.

## Commands

- Targeted test: `<exact command and example selector>`
- Full tests: `<exact command>`
- Format check/fix: `<exact commands>`
- Lint/static analysis: `<exact command>`
- Type check: `<exact command or "not applicable">`
- Build: `<exact command>`
- Smoke test: `<exact command>`

## Change rules

- <generated-code, migration, dependency, and compatibility rules>
- Answer/review/diagnose requests are read-only unless implementation is requested.
- For requested changes, make in-scope edits and run relevant validation.
- Ask before destructive operations, external writes, new production dependencies,
  or a material expansion of scope.

## Definition of done

- Run targeted validation for changed behavior.
- Run broader checks required by the affected package or CI.
- Report commands run, results, and any validation that could not run.

## Agent skills

<generated by setup-matt-pocock-skills; do not duplicate it>
```

Add nested `AGENTS.md` or `AGENTS.override.md` files only for genuinely different package/service rules. Codex stops its walk at the current working directory, and an override in a directory wins over a regular `AGENTS.md` in that same directory. [OpenAI: layering project instructions](https://developers.openai.com/codex/agent-configuration/agents-md#layer-project-instructions)

Verify discovery after edits with a new Codex session, for example:

```powershell
codex --ask-for-approval never "Summarize the current instructions."
codex --cd path\to\subtree --ask-for-approval never "Show which instruction files are active."
```

OpenAI documents those checks and notes that restarting the session rebuilds the chain; there is no instruction cache to clear manually. [OpenAI: verify and troubleshoot AGENTS.md](https://developers.openai.com/codex/agent-configuration/agents-md#verify-your-setup)

## 3. Give Codex a reproducible environment and fast feedback loops

The repository should expose stable, non-interactive commands that humans, CI, and Codex all use. Prefer one bootstrap entry point and one validation entry point per concern instead of asking the agent to reconstruct setup from prose. Make setup safe to rerun, pin toolchain/dependency versions using the project's normal lock/version files, and keep secrets out of command output. These are implementation recommendations; the key Codex requirement is that `AGENTS.md` names the exact commands the agent can execute and validate.

For Codex in the ChatGPT desktop app, **local environments** can define setup scripts for new worktrees and top-bar actions such as Run, Build, or Test. The app stores this configuration in the root `.codex` folder, and the generated file can be committed to share it. Setup runs when a new worktree is created; platform-specific setup/action variants are supported. Local environments are an app feature, not a replacement for repository scripts used by the CLI, IDE, or CI. [OpenAI: Local environments](https://developers.openai.com/codex/environments/local-environment)

Recommended arrangement:

- Repository command: installs dependencies/tools and performs any required initial generation/build.
- Local-environment setup: calls that repository command.
- Local actions: call the same repository test/build/run commands documented in `AGENTS.md`.
- CI: calls the same validation commands, possibly with a broader/full mode.

For **Codex cloud**, configure a setup script in the environment settings. Setup has internet access and runs before the agent; an optional maintenance script runs when a cached environment resumes. Setup runs in a separate Bash session, so a shell `export` does not persist into the agent phase. Environment variables remain available for the chat, while secrets are available to setup but removed before the agent phase. `AGENTS.md` remains where cloud agents discover project-specific lint and test commands. [OpenAI: Cloud environments](https://developers.openai.com/codex/environments/cloud-environment)

A useful validation ladder is:

1. The smallest targeted test that exercises changed behavior.
2. Formatting and lint/static analysis for touched code.
3. Type checking, if the language/project has a separate type-check step.
4. The affected package/module build.
5. A minimal runtime smoke test.
6. Full repository validation when required by risk or CI policy.

Document which steps are cheap/mandatory and which are expensive/conditional. This gives Sol explicit feedback without making every small change pay the cost of the entire suite. OpenAI specifically recommends targeted tests, applicable type/lint checks, affected-package builds, and a minimal smoke test, with an explanation when validation cannot run. [OpenAI: prompt the model to check its work](https://developers.openai.com/api/docs/guides/latest-model)

## 4. How globally installed Matt Pocock skills behave in Codex

Codex supports skills from repository, user, admin, and system locations. Global/user skills live in `$HOME/.agents/skills`; repository skills live under `.agents/skills` from the current working directory up to the repository root. Codex can invoke a skill implicitly from its description or explicitly; in the CLI/IDE, use `/skills` to browse or type `$` to mention one. [OpenAI: Build skills](https://developers.openai.com/codex/build-skills#where-codex-loads-local-skills)

Matt's repository explicitly supports Codex through the Agent Skills installer and says the skills work with any model. Its native Codex plugin is still described as future work, so the installed skill directories are the current integration. Sol needs no Matt-specific model adaptation. [mattpocock/skills README](https://github.com/mattpocock/skills/blob/main/README.md#L13-L51)

The global installation supplies the workflows, but **Matt's engineering configuration has no global mode**. Every repository needs its own committed `docs/agents/` files because downstream skills read them at runtime. [Matt's setup guide: prerequisites](https://github.com/mattpocock/skills/blob/main/docs/engineering/setup-matt-pocock-skills.md#L1-L21)

Run this explicitly from the repository root:

```text
$setup-matt-pocock-skills
```

Matt's documentation uses slash-command notation, but Codex's current documented explicit syntax is `$skill-name` (or selection through `/skills`). The setup skill is deliberately non-implicit, so Codex will not decide to run it automatically. [OpenAI: explicit skill invocation](https://developers.openai.com/codex/build-skills#how-chatgpt-and-codex-use-skills) [Matt setup skill metadata](https://github.com/mattpocock/skills/blob/main/skills/engineering/setup-matt-pocock-skills/agents/openai.yaml#L1-L5)

## 5. Exactly what `$setup-matt-pocock-skills` does

It is an interactive, prompt-driven repository bootstrap, not a deterministic script. It first inspects:

- `git remote -v` and `.git/config`.
- Root `AGENTS.md` and `CLAUDE.md`, including an existing `## Agent skills` section.
- `CONTEXT.md`, `CONTEXT-MAP.md`, root and package ADR directories.
- Existing `docs/agents/` and `.scratch/` conventions.
- Whether the `triage` skill is installed.
- Monorepo signals such as workspace configuration or populated packages.

It presents the findings, asks for the choices it cannot safely infer, shows the proposed instruction block and config files, and waits for confirmation before writing. [Matt setup SKILL.md: explore and confirm](https://github.com/mattpocock/skills/blob/main/skills/engineering/setup-matt-pocock-skills/SKILL.md#L13-L59)

It configures exactly three concepts:

| Concept | Choices/default |
| --- | --- |
| Issue tracker | GitHub via `gh`, GitLab via `glab`, local Markdown under `.scratch/<feature>/`, or another workflow described in prose |
| Triage labels | Only when `triage` is installed; defaults to `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix` |
| Domain docs | Single-context by default (`CONTEXT.md` plus `docs/adr/`); multi-context is offered only when monorepo signals exist |

[Matt setup SKILL.md: the three decisions](https://github.com/mattpocock/skills/blob/main/skills/engineering/setup-matt-pocock-skills/SKILL.md#L32-L49)

After confirmation it writes or updates:

- `docs/agents/issue-tracker.md`.
- `docs/agents/domain.md`.
- `docs/agents/triage-labels.md` only when `triage` is installed.
- One `## Agent skills` block in the selected root instruction file.

These files are configuration consumed by the other engineering skills; installed `SKILL.md` files should remain unchanged. Day-to-day adjustments can be made directly in `docs/agents/*.md`. Re-run setup to change trackers, restart the configuration, or reconcile a materially changed template after an upgrade. [Matt setup guide: output and lifecycle](https://github.com/mattpocock/skills/blob/main/docs/engineering/setup-matt-pocock-skills.md#L15-L21) [Matt setup guide: working state](https://github.com/mattpocock/skills/blob/main/docs/engineering/setup-matt-pocock-skills.md#L84-L94)

It does **not**:

- Install project dependencies or configure Codex's model/environment.
- Create actual GitHub/GitLab labels; `triage-labels.md` only maps canonical roles to existing label strings.
- Eagerly create `CONTEXT.md` or ADR content; the domain layout is recorded and `domain-modeling` fills it lazily as terminology or decisions emerge.
- Provide global configuration shared by every repository.

[Matt setup guide: labels and domain docs](https://github.com/mattpocock/skills/blob/main/docs/engineering/setup-matt-pocock-skills.md#L48-L74)

### Codex-specific `CLAUDE.md` caveat

The setup skill currently edits an existing `CLAUDE.md` before considering `AGENTS.md`. Upstream documents this as a known gap because Codex may not read the resulting block. For a Codex-first repository:

- If neither file exists, choose `AGENTS.md` when setup asks.
- If only `CLAUDE.md` exists, either make `AGENTS.md` canonical and reduce `CLAUDE.md` to a pointer, or configure `CLAUDE.md` as a Codex fallback filename where no `AGENTS.md` exists.
- If both exist and setup writes the block to `CLAUDE.md`, move the `## Agent skills` block into the root `AGENTS.md` so Codex is guaranteed to see it.

[Matt setup guide: known Codex gap](https://github.com/mattpocock/skills/blob/main/docs/engineering/setup-matt-pocock-skills.md#L48-L50) [OpenAI: fallback filenames and precedence](https://developers.openai.com/codex/agent-configuration/agents-md#customize-fallback-filenames)

## 6. Applied to this workspace

At research time, this repository contained only `README.md` and `LICENSE`; it had no root `AGENTS.md`/`CLAUDE.md`, no `CONTEXT*`, `docs/`, `.scratch/`, `go.mod`, `go.work`, documented setup/validation commands, or monorepo signals. Its origin is `https://github.com/ecedavis/golang.git`. The globally installed set includes `triage` and `setup-matt-pocock-skills`.

The existing user-level Codex config already selects `model = "gpt-5.6-sol"` with `model_reasoning_effort = "xhigh"`; it also sets the personal defaults `approval_policy = "never"` and `sandbox_mode = "workspace-write"`. This repository therefore does not need a `.codex/config.toml` merely to obtain Sol or those user-level execution defaults. Add a project config only if pinning choices for everyone is an intentional repository policy.

If setup runs here now, the expected path is:

1. Explicitly invoke `$setup-matt-pocock-skills` from the repository root.
2. Choose `AGENTS.md` when asked which instruction file to create.
3. Confirm GitHub as the issue tracker if this repository uses its origin's GitHub Issues; setup should propose it from the remote.
4. Accept or map the five triage labels; because `triage` is installed, the label section will run.
5. Accept single-context domain docs unless this repository later becomes a genuine monorepo.
6. Review the draft, then allow it to write the root block and `docs/agents/*.md`.
7. Add real, verified setup/build/test/lint commands to `AGENTS.md` once the project has a toolchain. Do not invent placeholder commands and present them as runnable.
8. Start a new Codex session and ask it to summarize active instructions; check `/skills` and explicitly mention one installed skill to verify discovery.

No project runtime code or configuration was changed by this research. The recommended `AGENTS.md`, `.codex/config.toml`, environment setup, and Matt per-repo bootstrap remain proposed next steps rather than applied changes.
