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


