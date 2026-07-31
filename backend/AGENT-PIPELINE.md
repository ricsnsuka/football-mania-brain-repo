#  Football Management System Agent Ecosystem

This directory contains specialized AI agents that manage different phases of the development lifecycle.
Each agent is a focused expert in their domain, working together to maintain code quality, 
architecture compliance, and comprehensive documentation.

---

## 🎯 Start Here — Orchestrator

> **New to the agent ecosystem?** Always start with the **Orchestrator**.
> It classifies your prompt, builds the optimal pipeline, and delegates to the right specialists automatically.

| Agent | Purpose | File |
|-------|---------|------|
| 🎯 **Orchestrator** | **Entry point** — classifies any prompt and delegates to the right pipeline | `orchestrator.agent.md` |

---

##  Quick Agent Reference

| Agent | Purpose | When to Use | File |
|-------|---------|------------|------|
| 🎯 **Orchestrator** | Classify prompt, build & execute optimal pipeline | **Always — start here** | `orchestrator.agent.md` |
|  **Requirements Analyst** | Break down requirements, analyze impact | New feature request | `requirements-analyst.agent.md` |
|  **API Designer** | Design API contracts & DTOs | Planning endpoints | `api-designer.agent.md` |
| ️ **DB Migration Engineer** | Write Flyway migrations | Database schema changes | `db-migration.agent.md` |
| ⚙️ **Dev Assistant** | Implement features end-to-end | Code implementation | `dev-assistant.agent.md` |
| ️ **Phase 3 Compliance** | Enforce architecture patterns | Architecture reviews | `phase3-compliance.agent.md` |
|  **Test Engineer** | Write & maintain tests | Test coverage & fixes | `test-engineer.agent.md` |
|  **Security Auditor** | Review security & dependencies | Security reviews, CVEs | `security-auditor.agent.md` |
|  **Documentation Writer** | Create & maintain docs | Feature documentation | `documentation-writer.agent.md` |
| ️ **Documentation Organizer** | Audit & organize docs structure | Documentation cleanup | `documentation-organizer.agent.md` |
|  **Version Updater** | Manage releases & versions | Release preparation | `version-updater.agent.md` |
|  **Deployment Engineer** | Deploy to production | Deployment tasks | `deployment-engineer.agent.md` |
| ✦ **Calculation Service** | Refactor calculation logic | Calculation refactoring | `calculation-service.agent.md` |
| 📬 **Postman Engineer** | Create & update versioned Postman collections | API release, new/changed endpoints | `postman-engineer.agent.md` |

---

##  Development Lifecycle Phases

The agents follow a structured phase-based pipeline:

```
 Phase 1          Phase 2         ️ Phase 3         ⚙️ Phase 4
Requirements  →    API Design    →    DB Migration  →    Implementation
requirements-      api-designer       db-migration        dev-assistant
analyst

️ Phase 5          Phase 6          Phase 7          Phase 8
Architecture  →    Testing       →    Security      →    Documentation
Compliance                            & Deps
phase3-            test-engineer      security-           documentation-
compliance                            auditor             writer

 Phase 9          Phase 10
Version &    →    Deployment
Release
version-           deployment-
updater            engineer
```

**Cross-Cutting Agents:**
- `calculation-service` — Invoked by Phase 4 & 5
- `documentation-organizer` — Works in parallel with Phase 8
- `postman-engineer` — Invoked after any API release or endpoint change

---

## 🎯 Quickstart — Use the Orchestrator

The **Orchestrator** is the recommended entry point for all tasks.
It classifies your prompt and delegates to the correct specialist pipeline automatically:

```
run_subagent(
  agentName: "orchestrator",
  description: "Classify and delegate the task",
  task: "<your prompt here>"
)
```

The orchestrator will:
1. Classify the intent (new feature / bug fix / release / docs / etc.)
2. Determine which agents to run and in what order
3. Optimise the pipeline (skip unnecessary phases, parallelise independent tasks)
4. Delegate to each specialist agent with full context carry-forward
5. Return a consolidated summary of everything done

---

##  How to Use Agents

### Starting an Agent

To invoke an agent directly (bypassing the orchestrator), use the `run_subagent` tool:

```
 Invoke an agent the automatic way:
run_subagent(
  agentName: "documentation-organizer",
  description: "Organize and clean up documentation",
  task: "Audit the current docs/ structure and organize..."
)
```

### Example Use Cases

#### Scenario 1: Implement a New Feature

1. **Start:** `requirements-analyst`
   ```
   "Analyze this new feature request and break it down into implementation tasks"
   ```

2. **Then:** `api-designer`
   ```
   "Design the API endpoints and DTOs for this feature"
   ```

3. **Then:** `db-migration` (if needed)
   ```
   "Create Flyway migrations for the new tables/columns"
   ```

4. **Then:** `dev-assistant`
   ```
   "Implement the feature following the API design"
   ```

5. **Then:** `calculation-service` (if calculations involved)
   ```
   "Refactor any calculations to use CalculationService"
   ```

6. **Then:** `phase3-compliance`
   ```
   "Review the implementation for architecture compliance"
   ```

7. **Then:** `test-engineer`
   ```
   "Write comprehensive tests to reach 90% coverage"
   ```

8. **Then:** `security-auditor`
   ```
   "Review security implications and scan dependencies"
   ```

9. **Then:** `documentation-writer`
   ```
   "Write API docs, feature guides, and endpoint changes"
   ```

10. **Then:** `documentation-organizer`
    ```
    "Ensure the new docs are properly organized by version/type"
    ```

#### Scenario 2: Emergency Bug Fix

1. **Direct:** `dev-assistant`
   ```
   "Fix the bug described in issue #123 in PlayerController"
   ```

2. **Then:** `test-engineer`
   ```
   "Add/update tests to prevent regression of this bug"
   ```

3. **Then:** `documentation-writer`
   ```
   "Document the fix in release notes if user-facing"
   ```

#### Scenario 3: Release Preparation

1. **Start:** `version-updater`
   ```
   "Prepare for release v5.1.0 - bump versions, validate dependencies"
   ```

2. **Then:** `deployment-engineer`
   ```
   "Deploy to staging and run smoke tests"
   ```

#### Scenario 4: Documentation Cleanup

1. **Direct:** `documentation-organizer`
   ```
   "Audit docs/ folder and reorganize by version/type/functionality"
   ```

---

## ️ Documentation Organization

The new **Documentation Organizer** agent handles:

- **Audit**: Find orphaned files, duplicates, outdated docs
- **Organization**: Arrange docs by version, type, and feature
- **Consolidation**: Merge duplicate documentation
- **Archive**: Move outdated files to `docs/archive/`
- **Indexing**: Maintain central documentation indices
- **Validation**: Verify no broken links, consistent formatting

### Documentation Structure (Organized)

```
docs/
├── INDEX.md                       ← Master entry point
├── VERSION_MAP.md                 ← Feature-to-version mapping
├── vX.Y.Z/                        ← Versioned documentation
│   ├── README.md
│   ├── RELEASE_NOTES.md
│   ├── api/, implementation/, deployment/
├── features/                      ← Feature guides (cross-version)
├── technical/                     ← Architecture deep-dives
├── guides/                        ← How-to guides
├── api/                          ← Current API reference
├── deployment/                    ← Deployment docs
├── testing/                      ← Test guides
├── security/                     ← Security docs
├── database/                     ← DB documentation
├── next-release/                 ← Staging for next release
├── archive/                      ← Outdated docs
└── ...
```

---

##  Agent Coordination Rules

### Rule 1: Sequential Pipeline for Features

When implementing features, follow phases 1-8 sequentially:
- **Requirements** → **Design** → **Migration** → **Implementation** → **Compliance** → **Testing** → **Security** → **Documentation**

### Rule 2: Documentation Always Follows Feature Work

`documentation-writer` must run AFTER dev-assistant completes:
- Ensures docs reflect actual implementation
- Prevents documentation drift

### Rule 3: Organization Happens After Documentation

`documentation-organizer` runs after documentation-writer:
- Cleans up new docs into proper structure
- Maintains central indices
- Archives outdated docs

### Rule 4: Version Updates on Release

`version-updater` runs before `deployment-engineer`:
- Updates version numbers across all files
- Validates dependencies
- Creates release documentation

### Rule 5: Cross-Cutting Agents as Needed

`calculation-service` is called ad-hoc by dev-assistant or phase3-compliance:
- Not sequential
- Called when calculations are involved

---

##  Agent Specialization

Each agent is optimized for a specific domain:

### Phase 1: Requirements Analyst
- Breaks down requirements
- Analyzes impact on codebase
- Creates task checklist
- Identifies blockers

### Phase 2: API Designer
- Designs RESTful endpoints
- Creates DTO schemas
- Validates against business rules
- Generates OpenAPI specs

### Phase 3: DB Migration Engineer
- Writes Flyway SQL scripts
- Ensures H2 test compatibility
- Handles schema evolution
- Documents migration procedure

### Phase 4: Dev Assistant
- Implements features end-to-end
- Writes production code
- Handles integration between components
- Orchestrates other agents as needed

### Phase 5: Phase 3 Compliance
- Enforces CalculationService usage
- Validates architecture patterns
- Reviews code structure
- Ensures deprecation paths

### Phase 6: Test Engineer
- Writes unit tests
- Writes controller tests
- Writes integration tests
- Writes E2E tests
- Achieves 90% coverage

### Phase 7: Security Auditor
- Scans CVEs in dependencies
- Reviews authorization
- Checks OWASP compliance
- Validates secure patterns

### Phase 8: Documentation Writer
- Creates feature guides
- Writes API reference
- Documents endpoint changes
- Creates release notes

### Cross-Cutting: Documentation Organizer
- Audits documentation structure
- Organizes by version/type/feature
- Consolidates duplicates
- Maintains central indices

### Phase 9: Version Updater
- Bumps version numbers
- Validates dependencies
- Updates CHANGELOG
- Creates release docs

### Phase 10: Deployment Engineer
- Deploys to staging/production
- Runs smoke tests
- Creates deployment guides
- Handles rollback procedures

### Cross-Cutting: Postman Engineer
- Creates versioned Postman collection JSONs
- Produces three environment files (Local, Staging, Production)
- Keeps POSTMAN_COLLECTION_CHANGELOG.md up-to-date
- Writes per-version usage guides
- Validates 100% endpoint coverage against controllers

### Cross-Cutting: Calculation Service
- Refactors calculation logic
- Ensures CalculationService usage
- Validates calculation accuracy
- Optimizes performance

---

## ⚙️ Configuration

### Agent Tools

Each agent has access to:
- `read_file`, `create_file`, `open_file` — File operations
- `grep_search`, `semantic_search`, `file_search` — Code search
- `insert_edit_into_file`, `replace_string_in_file` — Code edits
- `run_in_terminal`, `get_terminal_output` — Terminal commands
- `get_errors` — Compilation/lint errors
- `list_dir` — Directory listing
- `validate_cves` — CVE validation (Security Auditor)

### Agent Constraints

- Each agent focuses on ONE specific phase/domain
- No agent deletes production code (only archives)
- All changes must be validated (tests, linting, builds)
- Documentation is NEVER deleted, only archived
- Cross-references between docs must be maintained

---

##  Best Practices

### When Using Agents

✅ **Do:**
- Be specific in your agent task description
- Provide context about what you're trying to achieve
- Let agents validate their own work before moving to the next phase
- Reference existing documentation when applicable
- Follow the phase pipeline for new features

❌ **Don't:**
- Skip phases (e.g., jumping from requirements to implementation)
- Use the wrong agent for a task (e.g., dev-assistant for test writing)
- Interrupt an agent mid-task
- Make manual edits that conflict with agent work
- Bypass security or compliance reviews

### Monitoring Agent Progress

Each agent provides:
- Clear task breakdown
- Step-by-step execution
- Validation at each step
- Error handling and recovery
- Summary of changes made

### Rollback If Needed

If an agent makes incorrect changes:
1. Note the specific changes
2. Request the agent to revert
3. Or manually revert and explain the issue
4. Re-run the agent with corrected instructions

---

##  Learning More

For detailed information about each agent, see:

- `requirements-analyst.agent.md` — Requirement analysis workflows
- `api-designer.agent.md` — API design patterns
- `db-migration.agent.md` — Migration strategies
- `dev-assistant.agent.md` — Implementation approach
- `phase3-compliance.agent.md` — Architecture rules
- `test-engineer.agent.md` — Testing strategies
- `security-auditor.agent.md` — Security practices
- `documentation-writer.agent.md` — Documentation standards
- `documentation-organizer.agent.md` — Documentation organization
- `version-updater.agent.md` — Release management
- `deployment-engineer.agent.md` — Deployment procedures
- `postman-engineer.agent.md` — Postman collection management
- `calculation-service.agent.md` — Calculation patterns

---

##  Contributing New Agents

To add a new agent:

1. Create `YOUR_AGENT_NAME.agent.md` in this directory
2. Follow the standard format (see any existing agent)
3. Define clear responsibilities and workflows
4. Include tool requirements and constraints
5. Update this README with the new agent
6. Update copilot-instructions.md if applicable

---

**Last Updated:** May 17, 2026  
**Maintained By:** AI Development Team  
**Version:** 2.1.0 — Postman Engineer added as cross-cutting agent
