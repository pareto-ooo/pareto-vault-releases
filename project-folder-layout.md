---
id: project-folder-layout
scope: global
severity: critical
schemaVersion: 1
---

# Project Structure

Every repository uses this top-level folder layout, unless it is a brown-field project with a structure of its own. Each concern gets one folder at the root, so the root itself stays clean. The detailed layout inside each folder is described in the standard that owns it and not repeated here.

```
<repository>/
├── code/
├── docs/
├── infrastructure/
└── pipeline/
```

| Folder | Contains | Owned by |
|--------|----------|----------|
| `code/` | Main code and test projects: solutions, backend projects, frontend apps and their tests | Coding standards (C#, Vue.js) |
| `docs/` | Project documentation: decisions, the error catalog, runbooks | This standard |
| `infrastructure/` | All infrastructure as code. terragrunt/terraform | Terraform standards |
| `pipeline/` | All pipeline assets: on Azure DevOps the entry points, `stages/`, `jobs/`, `steps/`, `scripts/`, `vars/`; on GitHub the `scripts/`, with workflows and actions under `.github/` where GitHub requires them | Azure DevOps or GitHub Actions standards |

## Rules

- **A folder only exists when it has content.** A repository without infrastructure has no `infrastructure/` folder. No empty placeholder folders.
- **Folder names are lowercase.** The same names in every repository, so paths, pipeline path filters and scripts work the same everywhere.
- **Tool configuration lives where the tool requires it.** A configuration file for a tool, scanner or extension is the one exception, and only when the tool cannot be pointed at the folder it belongs to. Configuration that a tool finds by searching upwards, such as `Directory.Build.props` and `global.json`, lives in `code/`.
- **Each deployable keeps everything it builds from inside `code/`.** Its project, tests, `Dockerfile` and `.dockerignore` sit together, so the build context never needs the repository root.
- **Database migrations are application code.** They live in the Infrastructure project under `code/`, not in `infrastructure/`. `infrastructure/` creates the database server and database, never the schema.
- **Generated output is never committed.** Build output, test results, coverage reports and generated documents such as the OpenAPI document are git-ignored.