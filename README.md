# **Architecture: Centralized CI/CD Management**

This project adopts a **Decoupled CI/CD Strategy** to maintain strict separation between business logic and infrastructure automation.

**Key Principles:**
*   **Infrastructure as Separate Code:** Workflow definitions, environment variables, and pipeline logic are maintained in a dedicated shared repository (`shared-ci-configs`). The application repository contains only a minimal bootstrap workflow that references this central source.
*   **Clean Release History:** By isolating CI/CD changes from application commits, release notes remain focused strictly on feature updates and bug fixes, free from infrastructure noise.
*   **Platform Agnosticism:** This structure facilitates easy migration between hosting platforms (e.g., GitHub Actions to GitLab CI). Switching platforms requires only updating the bootstrap reference or replacing it with a native config file, without altering core application code.
*   **Reusability & Consistency:** Standardized pipeline templates are enforced across multiple projects, ensuring consistent build, test, and deployment standards.

**Usage:**
The application repository relies on a lightweight `.github/workflows/bootstrap.yml` that triggers reusable workflows defined in the shared configuration repository. No complex pipeline logic resides directly in this codebase.

>**Platform Migration Note:**
> Because CI/CD logic is externalized, migrating this project to another platform (e.g., GitLab, Jenkins) involves only replacing the `.github/workflows/bootstrap.yml` file with a `.gitlab-ci.yml` or equivalent configuration. The application codebase remains untouched.

## Reusable Workflows

### `ccpp-autotools.yml` — build + run the test suite

Split into a `build` job and a `test` job (the build tree is archived and
passed between them). Beyond the plain `build-result`/`test-result`/
`unit-test-result`/`valgrind-result` outputs, it also exposes:

- `build-duration-seconds` / `test-duration-seconds` /
  `valgrind-duration-seconds` — wall-clock seconds for Configure+Compile,
  for `test-command` alone, and for `valgrind-command` alone, for
  consumers building a stage-timing summary. Toolchain installs and
  bootstrap are not counted. A duration is empty when its stage never
  started (setup failed, or Valgrind not enabled).
- `build-result` / `test-result` are `failure` when a setup step (apt,
  bootstrap, artifact download) fails and the build or tests never run,
  not just when they run and fail.
- `test-summary` — free-form Markdown, opt-in. If a consumer's
  `test-command` writes to the path in `$RICH_TEST_SUMMARY_FILE` (an env
  var this workflow sets before running it), that content is passed
  through verbatim. Consumers that don't use the convention just get an
  empty string — this workflow never inspects or parses `test-command`'s
  output itself, keeping any project-specific test-runner format out of
  the reusable layer.
- `valgrind-summary` — same convention, for `valgrind-command` via
  `$RICH_VALGRIND_SUMMARY_FILE`. The content is opaque here too, so a
  consumer wanting structured data back (e.g. JSON to build Checks API
  annotations from parsed leak reports) can write that instead of
  Markdown — this workflow doesn't care what's in the file.

### `ccpp-target-compile.yml` — compile a set of specific object targets

For canary-style checks (e.g. "does this still compile under flag set X")
rather than a full build + test cycle. Publishes a Markdown results table
to the job summary and a `results` JSON output
(`{name, label, target, outcome, blocking}` per target) — consumers
needing GitHub Check runs or other formatting should parse that JSON
rather than re-deriving results from logs.



## Versioning and pinning

Releases are tags on `main`:

- `v1.x.y` — immutable. A tag ruleset blocks moving and deleting them.
- `v1` — moves to the latest `v1.x.y`. Only maintainers may move it.

Within `v1`, input and output names and their meaning stay compatible;
a breaking change starts `v2`.

Pin consumers to a full commit SHA with the version as a comment, and let
Dependabot (`package-ecosystem: github-actions`) propose updates:

```yaml
uses: rdratlos/shared-ci/.github/workflows/ccpp-autotools.yml@<commit-sha> # v1.0.0
```

`@main` and `@v1` move without the consumer's CI ever testing the new
state. This repository pins its own actions by SHA too, so a consumer's
pin covers everything these workflows run.

### Release procedure

1. Merge the change into `main` with a merge commit, after a consumer
   (racoon-ipsec-tools) has run green against the pull request branch
   (`uses: ...@<branch>` in a draft pull request there).
2. Check that `main`'s tree equals what the consumer tested:
   `git diff <tested-branch-head> origin/main` must be empty. If not,
   test again.
3. Tag: `git tag -a v1.x.y -m "v1.x.y" origin/main && git push origin v1.x.y`.
4. Move consumers to the new SHA (Dependabot pull request, or the draft
   pull request from step 1). Their CI runs on the pinned SHA before it
   lands.
5. Then move `v1`: `git tag -f v1 v1.x.y^{commit} && git push -f origin v1`.

## Security notes for consumers

- **`*-command` inputs (and `env-vars`) are code.** They are executed with
  `eval`; the calling workflow file is the trust boundary. Never build one
  from untrusted context such as `github.head_ref`, PR titles or bodies —
  pass such values to your own steps through `env:` instead.
- **Outputs carry text the tested code can influence.** `test-summary`,
  `valgrind-summary` and `results` contain whatever the build/test wrote.
  Consume them through `env:` (`"$VAR"` in shell, `process.env.VAR` in
  `actions/github-script`), never as `${{ }}` inside a script.
- **`runs-on`:** these workflows build and run the checked-out code, which on
  a public repository includes pull requests from forks. Don't point them at
  self-hosted runners for such events.
