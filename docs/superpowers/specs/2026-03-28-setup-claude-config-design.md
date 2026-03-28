# Setup Claude Config — Design Spec

## Summary

A Claude Code custom slash command (`/setup-claude-config`) that automates the creation of full Claude configuration for any project. It runs as a conversational wizard inside Claude Code, auto-detecting the project's tech stack and generating `CLAUDE.md`, `.claude/settings.local.json`, and optionally `.claude/commands/`.

## Command Location

`/Users/kondetiudaykirangmail.com/IdeaProjects/Claude-Workspace/.claude/commands/setup-claude-config.md`

Invoked via `/setup-claude-config` from the Claude-Workspace project.

## Wizard Flow

### Step 1: Discover Projects

- Prompt user for a base path (default: `~/IdeaProjects/`)
- Scan the base path and list all subdirectories as a numbered menu
- User selects one project by number

### Step 2: Detect Tech Stack

Scan the selected project directory for the following indicators:

| File Found | Detected Stack | Bash Permissions |
|---|---|---|
| `package.json` | Node.js / JS/TS | `Bash(npm:*)`, `Bash(npx:*)` |
| `pom.xml` | Java / Maven | `Bash(mvn:*)` |
| `build.gradle` / `build.gradle.kts` | Java / Gradle | `Bash(gradle:*)`, `Bash(./gradlew:*)` |
| `requirements.txt` / `pyproject.toml` / `setup.py` | Python | `Bash(pip:*)`, `Bash(python:*)` |
| `Cargo.toml` | Rust | `Bash(cargo:*)` |
| `go.mod` | Go | `Bash(go:*)` |
| `Gemfile` | Ruby | `Bash(bundle:*)`, `Bash(rake:*)` |
| `.csproj` / `.sln` | C# / .NET | `Bash(dotnet:*)` |

Always included: `Bash(git:*)`.

Additionally detect:
- **Framework** — e.g., React/Next.js from `next` in package.json dependencies, Spring from pom.xml parent/dependencies
- **Test runner** — Jest, JUnit, pytest, cargo test, go test, etc.
- **Linter** — ESLint, Prettier, Checkstyle, pylint/ruff, clippy, etc.

Report findings to user before proceeding.

### Step 3: Preview Config

Present three sections for review:

#### 3a. CLAUDE.md

Generated from this template:

```markdown
# {Project Name}

## Overview
{Brief project description — from README if available, otherwise user provides}

## Tech Stack
- Language: {detected}
- Framework: {detected}
- Build tool: {detected}
- Test runner: {detected}
- Linter: {detected}

## Build & Run Commands
- Install: `{detected command}`
- Build: `{detected command}`
- Test: `{detected command}`
- Lint: `{detected command}`
- Run: `{detected command}`

## Project Structure
{Auto-generated top-level directory tree}

## Coding Conventions
- {Stack-appropriate defaults, e.g., camelCase for JS, snake_case for Python}
- {Framework-specific patterns}

## Rules
- Always run tests before committing
- Do not modify generated/build output directories
```

Each section is editable — user can tweak, add, or remove content.

#### 3b. .claude/settings.local.json

```json
{
  "permissions": {
    "allow": [
      "Bash(git:*)",
      "{stack-specific permissions...}"
    ]
  },
  "plugins": [
    "superpowers",
    "frontend-design",
    "docs",
    "pptx",
    "pdf",
    "code-review"
  ]
}
```

Note: The `plugins` array is adjusted based on user opt-out choices. If "remove all" is selected, the `plugins` key is omitted entirely.

#### 3c. Skills (enabled by default)

Default skills included:
- `superpowers`
- `frontend-design`
- `docs`
- `pptx`
- `pdf`
- `code-review`

Opt-out options:
- "Remove all" — disables all skills
- Remove individually — e.g., "remove pptx, pdf"
- "Keep all" — proceed with defaults

#### 3d. Custom Commands (opt-in)

Ask: "Generate stack-specific slash commands? (yes/no)"

If yes, generate `.claude/commands/` with:

**Universal commands (always included if opted in):**
- `run-tests.md` — Run the test suite
- `build.md` — Build the project

**Stack-specific commands:**

| Stack | Additional Commands |
|---|---|
| Node.js | `lint.md`, `dev-server.md` |
| Java/Maven | `lint.md`, `package.md` |
| Python | `lint.md`, `type-check.md` |
| Go | `lint.md`, `vet.md` |
| Rust | `lint.md` (clippy), `check.md` |

Each command is a markdown file with a prompt instructing Claude what to do, referencing the detected tool (e.g., "Run tests using Jest" or "Run tests using pytest").

### Step 4: Edit

Allow user to modify any section before generating. Accept free-form edits like:
- "Change the project description to X"
- "Add a rule: never use console.log"
- "Remove the lint command"

### Step 5: Generate

Write all files to the target project directory:
- `{project_path}/CLAUDE.md`
- `{project_path}/.claude/settings.local.json`
- `{project_path}/.claude/commands/*.md` (if opted in)

Confirm what was created with a summary listing all files and their paths.

## Tech Stack Detection Details

### Framework Detection (package.json example)

Scan `dependencies` and `devDependencies` for known frameworks:
- `next` → Next.js
- `react` → React
- `vue` → Vue.js
- `@angular/core` → Angular
- `express` → Express.js
- `nestjs` → NestJS

### Build Command Inference

| Stack | Install | Build | Test | Lint | Run |
|---|---|---|---|---|---|
| Node.js (npm) | `npm install` | `npm run build` | `npm test` | `npm run lint` | `npm start` |
| Node.js (yarn) | `yarn` | `yarn build` | `yarn test` | `yarn lint` | `yarn start` |
| Maven | — | `mvn clean install` | `mvn test` | — | `mvn spring-boot:run` |
| Gradle | — | `./gradlew build` | `./gradlew test` | — | `./gradlew bootRun` |
| Python (pip) | `pip install -r requirements.txt` | — | `pytest` | `ruff check .` | `python main.py` |
| Rust | — | `cargo build` | `cargo test` | `cargo clippy` | `cargo run` |
| Go | `go mod download` | `go build ./...` | `go test ./...` | `go vet ./...` | `go run .` |

### Coding Convention Defaults

| Language | Naming | Style |
|---|---|---|
| JavaScript/TypeScript | camelCase variables, PascalCase components | Prefer const, arrow functions |
| Java | camelCase variables, PascalCase classes | Follow project's existing style |
| Python | snake_case variables, PascalCase classes | PEP 8 |
| Go | camelCase unexported, PascalCase exported | gofmt |
| Rust | snake_case variables, PascalCase types | rustfmt |

## Non-Goals

- No browser-based UI — the wizard runs entirely within Claude Code
- No persistent config database — each run is independent
- No remote project support — local directories only
- No auto-updating of existing configs — generates fresh files (warns if files already exist)

## Edge Cases

- **Existing config files** — warn the user and ask whether to overwrite, merge, or skip
- **No detectable stack** — fall back to asking the user for language/framework manually
- **Multiple stacks in one project** — detect all and combine permissions (e.g., a monorepo with both Node.js and Python)
- **Empty project** — generate minimal CLAUDE.md with placeholder sections
