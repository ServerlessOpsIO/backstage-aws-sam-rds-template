# Backstage AWS SAM RDS Template Constitution

## Core Principles

### I. Every option combination renders correctly
Every combination of template options MUST render a complete, valid project.
A form choice MUST NOT produce a broken file. Content declared inside a
`{% if %}` block MUST NOT be referenced outside that block. Every variable used
by a skeleton, including in file and directory names, MUST be passed in the
`fetch:template` `values` for each template rendering it. JSON and YAML
skeletons MUST remain valid for every option combination, including conditional
commas and indentation. GitHub Actions expressions `${{ }}` in files also
rendered by the template engine MUST be enclosed in `{% raw %}...{% endraw %}`.
The generated project MUST pass its pipeline's validation, build, and test
steps without hand edits. Changes to a skeleton or option MUST be verified by
rendering every affected combination, not only the default.

### II. The catalog drives the inputs
Ownership and placement MUST come from catalog pickers configured with
`allowArbitraryValues: false`, filtered by the preceding field where applicable:
owner through `MyGroupsPicker`, then Domain, System, and Environment and cloud
account. Templates that operate on an existing component MUST select that
component from the catalog. A catalog entity MUST NOT be represented by a
free-text field. Values read from `catalog:fetch` MUST come from the entity
that owns them; changing the `entityRefs` order MUST update every result read by
index. Implicit catalog contracts MUST be documented next to their use. In
this template, the cloud-account entity's `metadata.title` is the AWS account
ID passed to publishing as `DEPLOYMENT_ACCOUNT_ID`. The current catalog-info
skeleton also constructs the owner and system references from fetched
`metadata.title` values, so it assumes those titles equal the corresponding
entity names unless the mapping is changed.

### III. Published repositories and entities are predictable
The repository name and generated Component name MUST be the same value:
the user's name input lowercased. The GitHub organization MUST come from the
hidden `github_org` parameter, defaulting to `ServerlessOpsIO`; it MUST NOT be
hard-coded in steps or skeletons. The generated `catalog-info.yaml` MUST set
`owner`, `system`, `lifecycle`, the `github.com/project-slug` annotation, and a
`spec.type` identifying the generated entity as `iac`. It MUST declare
`dependsOn` or `providesApis` relations for catalog entities the component
depends on or exposes.

Project templates MUST publish with `main` as the default branch, set repository
custom properties for domain, system, and owner from catalog selections, and
register `/catalog-info.yaml` at the repository root. Repository visibility,
branch protection, and review requirements MUST be consistent across project
templates; any difference MUST be explained in that template. This repository's
project template publishes public repositories and currently disables default
branch protection and required approving reviews.

### IV. Steps are ordered, optional, and fail safely
Project-template steps MUST follow this order: fetch entities; render
skeletons; render pipelines; add optional starters; publish; register; then run
optional steps that act on the published repository or external services.
Optional steps MUST be disabled unless selected, guarded with `if:`, and expose
and require their inputs only when selected. External-service steps MUST run
after registration so a failure leaves a registered project. Comments MUST
explain non-obvious behavior, including explicit values, external-resource
matching, or filter dependencies. This repository currently has no optional
post-publish steps.

### V. Credentials and secrets stay out of templates
Secrets and account-specific values MUST reach generated repositories only
through scaffolder secrets and the `secrets` input of `publish:github`. Secrets
MUST NOT be placed in skeleton files or template defaults, or committed to this
repository. Skeletons MUST refer to generated-project secrets by name for
pipelines to resolve at run time. Actions performed on a user's behalf MUST use
that user's GitHub credentials, requested through `SoRequestUserCredentials`
into `USER_GITHUB_TOKEN` with only the required scopes. Tokens MUST NOT be
written to files, git remotes, logs, or pull-request text. Commits MUST use the
template runner's catalog profile name and email.

### VI. The form is honest and templates are consistent
Parameter titles and descriptions MUST match field behavior, including hidden
fields, and defaults MUST be values users can accept as-is. Fields applicable
only to some choices MUST be shown and required only for those choices through
JSON Schema `dependencies`. An enum option MUST NOT be offered unless it renders
a working project. Step IDs and names MUST describe their actions. Templates
in this repository MUST share the same first parameter page, metadata
conventions (owner, tags, and description format), and `publish:github`
settings; changes to shared fields MUST be applied consistently or their
differences explained in the pull request. Each template's `output.links` MUST
link to every created repository or pull request, catalog entity, and optional
agent task or external resource.

## Repository scope and generated artifact boundaries
This repository currently contains one project template, `template.yaml`
(`aws-sam-rds-postgres`), which creates a Backstage `Component` with
`spec.type: iac`. Its rendered project is built from `skeleton/`, including
`skeleton/catalog-info.yaml` and the SAM `skeleton/template.yaml`. The
`pipeline/` directory contributes the `main` and `branch` GitHub Actions
workflows. Template rules govern form parameters, scaffolder steps, rendering,
publishing, registration, and optional actions. They MUST NOT prescribe the
infrastructure or application design of generated projects; those projects'
own constitutions govern their design. The optional `db_name` parameter
controls the conditional `DbName` parameter and cluster `DatabaseName`
property in `skeleton/template.yaml`; both MUST render together or be omitted
together.

## Development workflow and quality gates
Before merge, affected templates MUST be run through the Backstage template
editor dry run. Every affected option combination MUST be rendered and checked
against Principle I. Changes to publishing, registration, optional external
steps MUST be exercised by a real run against a test component or sandbox. The
pull request MUST list what was run.

Changes to template inputs, generated files, required catalog entities, required
secrets, required Backstage actions, or other prerequisites MUST update this
repository's `README.md` in the same change; update a generated project's
README when affected. A new or changed scaffolder action MUST be identified in
the README together with the Backstage module providing it.

Every new feature MUST be evaluated against the generated project's handbook
in `skeleton/docs/handbook/` before it is planned and again before merge. The
feature's specification or plan MUST record which handbook pages the feature
affects, or that it affects none. Affected pages MUST be updated in the same
change so the handbook describes the generated project as it will be rendered,
including every affected option combination. When no page needs updating, the
pull request MUST say so and why.

Generated, OS, and editor files such as `.DS_Store`, `__pycache__`, and
`node_modules` MUST NOT be committed. Work MUST use `feature/` or `bug/`
branches referencing an issue number where one exists. Commit messages MUST
use conventional-commit style such as `feat:`, `fix:`, or `refactor:`. Changes
MUST merge through pull requests.

## Governance
This constitution supersedes conflicting conventions in this repository.
Every specification, plan, and pull request MUST be checked for compliance.
Amendments MUST be made in the same change that introduces a new rule and MUST
include a Sync Impact Report for human review; remove that temporary report
before committing the amended constitution.

Versioning follows semantic versioning: MAJOR removes or incompatibly redefines
a principle; MINOR adds a principle or section, or materially expands guidance;
PATCH clarifies wording or makes a non-semantic refinement. An amendment MUST
update the last-amended date and version. If a change's compliance impact is
unclear, the pull request MUST explain the interpretation and rationale.

**Version**: 1.1.0 | **Ratified**: 2026-10-29 | **Last Amended**: 2026-10-29
