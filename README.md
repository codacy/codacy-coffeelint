# codacy-coffeescript

Offers coffeescript support for Codacy

[![Codacy Badge](https://api.codacy.com/project/badge/Grade/e3358fb3445a4337b9e20c5c09106925)](https://www.codacy.com/gh/codacy/codacy-coffeelint?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=codacy/codacy-coffeelint&amp;utm_campaign=Badge_Grade)
[![Build Status](https://circleci.com/gh/codacy/codacy-coffeelint.svg?style=shield&circle-token=:circle-token)](https://circleci.com/gh/codacy/codacy-coffeelint)

## Forked from [rrdelaney/codacy-coffeescript](https://github.com/rrdelaney/codacy-coffeescript)

A big thank you to Ryan Delaney for taking the time to implement this coffeescript solution. Show him your love @ https://github.com/rrdelaney.

## Developing

### Dependencies

There are some dependencies not installed by the bootstrap script. You will need

* JDK
* Scala
* sbt
* node
* npm

### Bootstrapping

Bootstrap the development environment with

```bash
sh script/bootstrap
```

### Testing
Test the project with

```bash
sh script/test
```

### Generate docs

Generate or update docs with

```bash
grunt patterns
```

## Agent Playbook: Updating This Repository End-to-End

This section is written for an AI coding agent (or a human) tasked with updating this repo — most commonly bumping the wrapped `@coffeelint/cli` version, but also base image / CircleCI orb bumps. Follow it top to bottom; it tells you what to change, how to regenerate derived files, how to test locally, and how to interpret CI so you can iterate on failures without guessing.

### 1. What this repository is

This is a **Codacy engine**: a small Node.js/CoffeeScript wrapper (`src/*.coffee`, compiled by Grunt into `target/*.js`) that packages [CoffeeLint](https://github.com/coffeelint/coffeelint) (`@coffeelint/cli` on npm) as a Docker image Codacy's platform can run against a customer's source code. Despite what the "Dependencies" section above says, **this is not a Scala/sbt project** — there is no Scala anywhere in this repo; build/lint/compile is entirely Grunt + npm (that stale requirement list should be ignored/fixed opportunistically, but is out of scope for a version-bump task).

The `docs/` directory is machine-consumed configuration, not just documentation:

- `docs/patterns.json` — the full list of CoffeeLint rules ("patterns") Codacy knows about, their parameters/defaults, and which are enabled by default. **Generated file, do not hand-edit.**
- `docs/description/description.json` + `docs/description/*.md` — human-readable titles/descriptions per pattern, used in the Codacy UI. **Generated file, do not hand-edit.**
- `docs/tests/*.coffee` and `docs/multiple-tests/config-patterns/*` — fixtures used by `codacy-plugins-test` to validate the engine actually produces the results it claims for real code samples.
- `docs/tool-description.md` — short blurb about the tool, hand-maintained.

All the generated artifacts above come from the Grunt task **`patterns`**, defined in `tasks/patterns.js` (registered via `Gruntfile.coffee`, part of the default `grunt` task). It requires an **`tasks/patterns.json` spec file** that must be **manually curated**: `tasks/patterns.js` only emits a pattern into `docs/patterns.json` if that rule name already has an entry (category/level/parameters) in `tasks/patterns.json` — otherwise it just logs `Missing rule: <name>` to the console and silently skips it. So on every CoffeeLint version bump you must check the generator's console output for `Missing rule:` lines and add the missing rules (category/level/parameters) to `tasks/patterns.json` by hand before re-running, and likewise remove entries for rules CoffeeLint no longer ships.

### 2. Files that encode versions — check all of these on every update

| File | What it controls | What to check |
|---|---|---|
| `package.json` → `dependencies["@coffeelint/cli"]` | The CoffeeLint version bundled and analyzed | Bump to the target version. `package.json` also pins `coffeescript` (the CoffeeScript compiler) — only touch if asked or if CoffeeLint's own `coffeescript` peer version changed. |
| `package-lock.json` | Locked resolved versions/integrity hashes for the above | Regenerate via `npm install` after bumping `package.json` — do not hand-edit. |
| `tasks/patterns.js` → `require('...')` | Which npm package is loaded to read `.VERSION`/`.RULES` at generation time | Historically this needed updating too (`require('coffeelint')` → `require('@coffeelint/cli')` when the package was renamed on npm — see commit `3b4a760`). Confirm it still requires the correct package name for the version you're bumping to. |
| `tasks/patterns.json` | Manually-curated spec of category/level/parameters per rule, consumed by the generator | Add entries for any new rules CoffeeLint introduced (watch generator output for `Missing rule:`), remove entries for rules it dropped, update parameters if CoffeeLint changed a rule's shape. |
| `Dockerfile` → `alpine:<version>` (both build and runtime stages) | Base OS image, which brings its own `nodejs`/`npm` | Bump both `FROM alpine:...` lines together. Only bump opportunistically or when asked/needed — not required for every CoffeeLint bump. |
| `.circleci/config.yml` → `codacy/base` orb, `codacy/plugins-test` orb | Shared CircleCI steps (checkout, versioning, docker build/publish, plugins-test run, tagging) | Check for newer published orb versions when asked to bump CI tooling; not tied to CoffeeLint bumps specifically. |

Look at prior bump commits for the exact shape of a real diff: `git log --oneline --all | grep -i bump`, then e.g. `git show 3b4a760` (a `@coffeelint/cli` 0.0.0→5.2.11-family bump) touches `package.json`/`package-lock.json`, `tasks/patterns.js`, `tasks/patterns.json`, `docs/patterns.json`, `docs/description/*`, the `.circleci/config.yml` orb version, and the `Dockerfile` alpine version, all in one commit; a fixup commit (`dd809e9`/`fabe273`, "Fix tests") followed to correct stale fixtures in `docs/tests/*.coffee` that the new lint behavior broke.

### 3. Step-by-step update procedure

1. **Bump `@coffeelint/cli` (and/or `coffeescript`)** in `package.json`, then run `npm install` to refresh `package-lock.json`.
2. **Check `tasks/patterns.js`** still requires the right package name for the version you bumped to (it must match the key you changed in `package.json`).
3. **Regenerate the docs:**
   ```bash
   npm install -g grunt-cli   # if not already available
   npm install
   grunt patterns
   ```
   Watch the console for `Missing rule: <name>` — for each one, add a corresponding entry (category/level/parameters) to `tasks/patterns.json`, then re-run `grunt patterns`. Also remove any `tasks/patterns.json` entries for rules the new version no longer ships (they'll otherwise keep appearing in `docs/patterns.json` even though CoffeeLint dropped them — verify against `coffeelint.RULES` for the new version). Review the diff on `docs/patterns.json` and `docs/description/*` for new/removed/renamed rules.
4. **Build and lint the CoffeeScript source, and compile it:**
   ```bash
   grunt        # runs clean:dist -> coffeelint:dist -> coffee:dist -> patterns (see Gruntfile.coffee)
   ```
   (this is also what `sh script/test` and the Dockerfile's builder stage do).
5. **Build the Docker image** (same as CI): `docker build -t codacy-coffeescript .`
6. **Run `codacy-plugins-test` locally** before pushing — clone https://github.com/codacy/codacy-plugins-test (as `script/bootstrap` does) and, per `script/test`, run:
   ```bash
   sbt "runMain codacy.plugins.DockerTest json codacy-coffeescript:latest"
   sbt "runMain codacy.plugins.DockerTest pattern codacy-coffeescript:latest"
   ```
   against your freshly built local image tag. There is also a `docs/multiple-tests/` fixture set (`config-patterns/patterns.xml` + `results.xml`) exercised by the `multiple` DockerTest command used in CI (`codacy_plugins_test/run` in `.circleci/config.yml`) — run that too if it applies to your change.
7. **Iterate on failures.** If a `docs/tests/*.coffee` fixture starts failing because the new CoffeeLint version changed a rule's real behavior (this has happened before — see commits `dd809e9`, `fabe273`), update the fixture to match the new, verified-correct output rather than fighting the tool.
8. **Commit** the version bump together with the regenerated `docs/` files and any `tasks/patterns.json` curation in one change.
9. **Push and open a PR.** CI (`.circleci/config.yml`) runs `codacy/checkout_and_version` -> `publish_local` (docker build) -> `plugins_test` (`codacy_plugins_test/run`) -> `publish_dockerhub` (master only) -> `tag_version`.
10. **Poll the PR's real CI checks until they all pass — local validation is NOT the finish line.** After every push, run `gh pr checks <pr-url>` and keep re-polling (short sleep while any check is `pending`) until all checks finish. If a check fails, fetch its actual log (the CircleCI job log linked from the check — don't guess), find the true root cause, fix it, push again (never `--no-verify`, never force-push), and re-poll. Repeat until every check is green. Only stop iterating when every check passes, or you hit a genuine product/infra decision that needs a human — in which case explain it in the PR rather than guessing.

### 4. Common failure modes and fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Generator logs `Missing rule: <name>` and the rule is absent from `docs/patterns.json` | New CoffeeLint version added a rule not yet in `tasks/patterns.json` | Add a category/level/(parameters) entry for it in `tasks/patterns.json`, re-run `grunt patterns` |
| A rule you expected still appears in `docs/patterns.json` after a bump that removed it upstream | `tasks/patterns.json` still has a stale manual entry | Remove that entry from `tasks/patterns.json`, re-run `grunt patterns` |
| `docs/tests/*.coffee` fixtures fail against the new image | CoffeeLint changed a rule's real detection behavior between versions | Update the fixture file to reflect the new, correct behavior (precedent: commits `dd809e9`, `fabe273`) |
| `grunt` fails requiring the coffeelint/cli package | `tasks/patterns.js`'s `require(...)` target doesn't match the package name/version now in `package.json` | Update the `require` string to match (precedent: commit `3b4a760` renamed `coffeelint` → `@coffeelint/cli`) |

### 5. Definition of done

- `@coffeelint/cli` (and/or `coffeescript`) version bumped in `package.json`, with `package-lock.json` regenerated via `npm install`.
- `tasks/patterns.js`'s `require(...)` target verified/updated to match.
- `tasks/patterns.json` manually curated for any rules added/removed upstream (no `Missing rule:` warnings left unresolved).
- `docs/patterns.json` and `docs/description/*` regenerated via `grunt patterns` and committed.
- `docs/tests/*` and `docs/multiple-tests/*` fixtures updated where new tool behavior made them stale.
- `grunt` (build + lint + compile) and `docker build` succeed locally.
- `codacy-plugins-test` `json`, `pattern`, and (if relevant) `multiple` DockerTest commands all pass locally against the freshly built image.
- **After pushing and opening/updating the PR, every CI check on it is green.** Poll `gh pr checks <pr-url>` and iterate on any failure (fetch the real CI log, fix, push, re-poll) until all pass — a passing local build is not sufficient.

## What is Codacy

[Codacy](https://www.codacy.com/) is an Automated Code Review Tool that monitors your technical debt, helps you improve your code quality, teaches best practices to your developers, and helps you save time in Code Reviews.

### Among Codacy’s features

- Identify new Static Analysis issues
- Commit and Pull Request Analysis with GitHub, BitBucket/Stash, GitLab (and also direct git repositories)
- Auto-comments on Commits and Pull Requests
- Integrations with Slack, HipChat, Jira, YouTrack
- Track issues in Code Style, Security, Error Proneness, Performance, Unused Code and other categories

Codacy also helps keep track of Code Coverage, Code Duplication, and Code Complexity.

Codacy supports PHP, Python, Ruby, Java, JavaScript, and Scala, among others.

### Free for Open Source

Codacy is free for Open Source projects.
