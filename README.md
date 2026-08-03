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

- `build-duration-seconds` / `test-duration-seconds` — wall-clock time for
  each job, for consumers building a stage-timing summary.
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


