# WIKI-FIRST TEMPLATE — Skill Creation Guide for a New Project

> **Purpose:** This document is a task for an AI agent (Code Agent) to set up a wiki-first skill in a new project.
> Copy this file into your project, fill in the parameters, and hand it to the AI agent for execution.

---

## INPUT DATA (FILL IN BEFORE USE)

| Parameter | Value |
|-----------|-------|
| `project_name` | Project name |
| `project_description` | Brief description — what the project does |
| `tech_stack` | Technologies (e.g. C#, .NET 8, gRPC, PostgreSQL) |
| `docs_path` | Path to documentation (e.g. `docs/`) |
| `wiki_path` | Path to wiki (e.g. `docs/wiki/`) |
| `skills_path` | Path to AI agent skills (e.g. `.qwen/skills/`) |
| `source_dirs` | Paths to source code directories |

---

## TASK FOR THE AI AGENT

### Goal

Create a **wiki-first system** for a new project — a set of wiki pages, skills, and instructions that will provide:

1. **Single source of truth** about the project in `docs/wiki/`
2. **Wiki-first rule** — all AI skills start with wiki before reading code
3. **Automatic freshness** — wiki is updated whenever code changes
4. **Context savings** — AI agent reads wiki (overview), not code (details)

---

### Step 1: Create wiki structure

**Task:** Create the wiki directory and initial files.

**Actions:**

```bash
mkdir -p {wiki_path}
```

**Create 3 files:**

| File | Purpose |
|------|---------|
| `{wiki_path}index.md` | Catalog of all wiki pages (navigation) |
| `{wiki_path}log.md` | Wiki change log (append-only) |
| `{wiki_path}Wiki Format.md` | Page format guide |

Below is the content for each file.

---

#### File 1: `wiki/log.md`

```markdown
# Wiki Log

Chronological journal of all wiki changes.

---

## [YYYY-MM-DD] setup | Wiki Initialization

**Type:** setup
**Author:** AI agent

### Description

Initial wiki structure created for project {project_name}.

### Created files

- `{wiki_path}index.md` — wiki page catalog
- `{wiki_path}log.md` — change log
- `{wiki_path}Wiki Format.md` — page format guide
```

---

#### File 2: `wiki/index.md`

```markdown
# Wiki Index

Catalog of all wiki pages for project {project_name}.

## 📋 Overview

| Category | Page | Description |
|----------|------|-------------|
| overview | [[Architecture]] | Overall system architecture |

## 📝 Wiki Instructions

| Page | Description |
|------|-------------|
| [[Wiki Format]] | Page format: frontmatter, links, log.md, index.md |

---

> **How to use:** When asked to analyse or study parts of the project, the LLM updates this index.md by adding new pages to the relevant categories.
```

---

#### File 3: `wiki/Wiki Format.md`

This file contains the wiki page format guide. Create it with the following content:

    ---
    tags: [wiki, format, instructions]
    created: YYYY-MM-DD
    updated: YYYY-MM-DD
    sources: [.qwen/skills/wiki-workflow/SKILL.md]
    ---

    # Wiki Format

    Guide to wiki page formats, links, and logs.

    ---

    ## Frontmatter

    Every wiki page starts with a YAML frontmatter:

        ---
        tags: [category, tag]
        created: YYYY-MM-DD
        updated: YYYY-MM-DD
        sources: [path/to/source1, path/to/source2]
        ---

    **Fields:**

    | Field | Description | Format |
    |-------|-------------|--------|
    | `tags` | Page categories | Array of tags |
    | `created` | Creation date | YYYY-MM-DD |
    | `updated` | Last update date | YYYY-MM-DD |
    | `sources` | Information sources | Array of paths |

    ---

    ## Links to wiki pages

    Use double square brackets:

        [[Architecture]] — overall system architecture
        [[Services]] — service descriptions

    ---

    ## Links to project files

    Relative paths from the wiki page:

        [Program.cs](../../src/Program.cs)

    ---

    ## Code quoting

    When referencing code, specify the file and context:

        From `Service.cs`:
        ```csharp
        public async Task Execute(CancellationToken ct) { ... }
        ```

    ---

    ## log.md format

    Each entry in `wiki/log.md`:

        ## [YYYY-MM-DD] type | Brief description

        **Type:** ingest | update | setup | lint
        **Author:** name

        ### Description

        What was done.

        ### Sources

        - `path/to/file1.cs`
        - `path/to/file2.md`

        ### What changed

        | Before | After |
        |--------|-------|
        | old | new |

    **Entry types:**

    | Type | Description |
    |------|-------------|
    | `setup` | Initial setup |
    | `ingest` | Adding new knowledge |
    | `update` | Updating existing pages |
    | `lint` | Fixing wiki issues |

    ---

    ## index.md format

    Catalog of wiki pages with categories:

        # Wiki Index

        ## 📋 Overview

        | Category | Page | Description |
        |----------|------|-------------|
        | overview | [[Page]] | Description |

    **Rules:**
    - Group by categories
    - Use emoji for visual separation
    - Every page must be in the catalog
    - Brief description for each page

---

### Step 2: Create wiki-workflow skill

**Task:** Create an AI skill for wiki management.

**File:** `{skills_path}wiki-workflow/SKILL.md`

Create the file with the following content:

    ---
    name: wiki-workflow
    description: Manages the project wiki system — ingest, query, lint operations with docs/wiki/. Use for learning the project, updating knowledge, or checking integrity.
    ---

    # Wiki Workflow

    ## Skill Purpose

    This skill is the **wiki system manager**. It manages project knowledge:
    - Ingest — extracting knowledge from code into wiki
    - Query — searching and synthesising knowledge from wiki
    - Lint — checking wiki integrity
    - Post-Change Lint — updating wiki after code changes

    **When to use:**
    - "Update wiki" or "study X" → **Ingest** operation
    - "How does X work?" or "tell me about Y" → **Query** operation
    - "Check wiki" or "find wiki issues" → **Lint** operation
    - After any code change → **Post-Change Lint** (mandatory)

    **When NOT to use:**
    - For writing code → use `code-contributor`
    - For architecture review → use `code-architecture`

    ---

    ## SOURCE PRIORITY (MANDATORY)

    This skill is the **wiki system manager**. All other skills must consult wiki before reading project code.

    ### Source hierarchy

    | Level | Source | Priority | Description |
    |-------|--------|----------|-------------|
    | 1 | `docs/wiki/` | **Highest** | Concrete project knowledge. Actual project rules. |
    | 2 | Project code | **Secondary** | Source files — only if wiki is insufficient. |

    **Rule:** If wiki describes one thing and code shows another, wiki reflects the actual state. Code is the source of details, wiki is the source of overview.

    ### Wiki = overview, Code = details

    Wiki pages contain an **overview** of structure, patterns, and components. Detailed information is in the code:

    | Wiki contains | Code contains |
    |---------------|---------------|
    | Which components exist | Implementation of each method |
    | Patterns and conventions | Usage examples |
    | Dependency overview | Interface signatures |

    **When ingesting:** don't copy details from code into wiki. Wiki = overview + link to file.
    **When querying:** if wiki doesn't have details — read the code.

    ---

    ## Skill Operations

    ### 1. Ingest — Adding Knowledge

    **Triggers:** "analyse X", "study Y", "update wiki", "add knowledge about Z"

    **Algorithm:**

    1. Determine analysis scope (which parts of the project to study)
    2. Read the relevant project files
    3. Extract key information:
       - Architecture and patterns
       - Dependencies and interfaces
       - Decisions and changes
    4. Create or update wiki pages in docs/wiki/
    5. Update wiki/index.md (add new pages to the catalog)
    6. Add an entry to wiki/log.md

    ### 2. Query — Knowledge Retrieval

    **Triggers:** "how does X work", "tell me about Y", "what does Z do", project questions

    **Algorithm:**

    1. Read wiki/index.md for navigation
    2. Find relevant wiki pages
    3. If needed, read project files for clarification
    4. Synthesise an answer with quotes and links to files
    5. If information is missing — run Ingest to create it

    ### 3. Lint — Wiki Health Check

    **Triggers:** "check wiki", "find issues", "check integrity"

    **Algorithm:**

    1. Read all wiki pages
    2. Check for:
       - Contradictions between pages
       - Stale claims (compare with project code)
       - Broken wiki page links [[Page]]
       - Broken project file links
       - Missing cross-references between related topics
       - Pages without frontmatter
       - Missing entries in log.md
    3. Propose fixes
    4. Upon confirmation — apply fixes

    ### 4. Post-Change Lint — Check After Code Changes

    **Triggers:** Runs after **any** creation, modification, or deletion of code files.

    **Algorithm:**

    1. Determine which wiki pages may contain stale information
    2. Read the relevant wiki pages
    3. Read the changed project files
    4. Check:
       - Wiki descriptions match current code
       - Interfaces, methods, DTOs haven't changed
       - New files/classes are reflected in wiki
       - Deleted files are removed from descriptions
    5. Update wiki pages if needed
    6. Add an entry to wiki/log.md

    ---

    ## Prohibitions

    - Do NOT modify project files through wiki operations
    - Do NOT delete wiki pages without user confirmation
    - Do NOT clear `wiki/log.md` (append-only)
    - Do NOT rename wiki pages without updating all links

---

### Step 3: Create code-contributor skill

**Task:** Create an AI skill for writing code with the wiki-first rule.

**File:** `{skills_path}code-contributor/SKILL.md`

Create the file with the following content:

    ---
    name: code-contributor
    description: Helps add and modify project code according to established standards. Use when creating features, fixing bugs, refactoring, or writing documentation.
    ---

    # Code Contributor

    ## SOURCE PRIORITY (MANDATORY)

    For **any** request to write code, refactor, fix bugs, or create features:

    1. **First** read `docs/wiki/index.md`
    2. **Find** relevant wiki pages and study their content
    3. **Use wiki as the primary source** of architectural decisions, patterns, and conventions
    4. **Only if** wiki lacks the needed information — read project files
    5. **After** studying project code — update wiki if new or changed information is found

    **Wiki is the primary source of knowledge, not the project code.**

    ### Wiki vs Code — Where to Find What

    | Information type | Where to look |
    |------------------|---------------|
    | Architecture, patterns, conventions | **Wiki** — overview and links |
    | Implementation details | **Code** — source files |
    | Which tests exist | **Wiki** |
    | What each test method tests | **Code** — test files |

    ---

    ## Skill Purpose

    This skill is the **developer's workbench**. Contains rules for writing code according to project standards.

    **When to use:**
    - Creating new features or services
    - Fixing bugs
    - Refactoring existing code
    - Writing project documentation

    **When NOT to use:**
    - For architecture review → use `code-architecture`
    - For wiki management → use `wiki-workflow`

    ---

    ## Before Starting Work

    Determine the task type and read the relevant wiki pages:

    | Task type | Read in wiki |
    |-----------|-------------|
    | New component | Architecture page, patterns |
    | Code change | Code patterns |
    | Database work | Database page, data access patterns |
    | API contract | Contracts page |

---

### Step 4: Create code-architecture skill

**Task:** Create an AI skill for architecture review with the wiki-first rule.

**File:** `{skills_path}code-architecture/SKILL.md`

Create the file with the following content:

    ---
    name: code-architecture
    description: Checks code architecture for SOLID compliance, DI, and project standards. Use during code review, class creation, module design, or solution analysis.
    ---

    # Code Architecture

    ## SOURCE PRIORITY (MANDATORY)

    For **any** request for architectural analysis, code review, class creation, or module design:

    1. **First** read `docs/wiki/index.md`
    2. **Find** relevant wiki pages and study them
    3. **Use wiki as the primary source** of architectural rules, patterns, and conventions
    4. **Only if** wiki lacks the needed information — read project files
    5. **After** analysis — provide a review with specific findings and recommendations

    **Wiki is the primary source of knowledge.**

    ---

    ## Skill Purpose

    This skill is the **architectural reviewer**. It evaluates code for compliance with SOLID principles, project patterns, DI standards, and layered architecture.

    **When to use:**
    - Code review
    - Evaluating architectural decisions before implementation
    - Checking DI configuration correctness
    - Analysing component relationships

    **When NOT to use:**
    - For writing new code → use `code-contributor`
    - For bug fixes → use `code-contributor`

---

### Step 5: Create QWEN.md in project root

**Task:** Create the main rules file for the AI agent with the wiki-first rule.

**File:** `QWEN.md` (project root)

Create the file with the following content:

    # Qwen Code — Project Rules

    ## WIKI-FIRST RULE

    For **any** work with the project (coding, review, refactoring, new features, bug fixes) the agent **must** start by studying the wiki.

    ### Procedure

    0. **Choose a skill** — determine the task type and use the corresponding skill:

       | Situation | Skill |
       |-----------|-------|
       | Write new code, feature, bug fix | `code-contributor` |
       | PR review, architecture check, design assessment | `code-architecture` |
       | Update wiki, find project information | `wiki-workflow` |

    1. **Wiki** — first read `docs/wiki/index.md`, find relevant pages, and use them as the **primary** source of knowledge about the project.
    2. **Code** — only if wiki is insufficient, read source files, documentation, configuration.
    3. **Update wiki** — after working with code, update the relevant wiki pages and add an entry to `docs/wiki/log.md`.

    ### Why

    The wiki contains condensed, structured project knowledge. Without wiki-first, the agent studies code from scratch every time, wasting context and time.

    ---

    ## CODE VERIFICATION

    After writing or modifying code, the agent **must** open the relevant wiki pages and verify the result against the rules and conventions described there.

    ---

    ## Project Skills

    Three skills are available in `{skills_path}/` — all contain the wiki-first rule:

    | Skill | Purpose | When to use |
    |-------|---------|-------------|
    | `code-contributor` | Writing and modifying code to standards | Creating features, bug fixes, refactoring |
    | `code-architecture` | Architecture review for SOLID, DI, standards | Code review, design assessment, module design |
    | `wiki-workflow` | Wiki management — ingest, query, lint | Updating wiki, finding information, integrity checks |

    **Responsibility split:**
    - `code-contributor` — **writes** code
    - `code-architecture` — **reviews** code
    - `wiki-workflow` — **manages** wiki
    - `code-contributor` references `code-architecture` for architectural rules
    - `code-contributor` references `wiki-workflow` for wiki page format

    **Do not use skills outside their intended purpose.**

    ---

    ## Wiki

    Instructions for working with wiki are in the skills.

    Wiki structure:

        docs/
        └── wiki/
            ├── index.md     # Catalog
            ├── log.md       # Change log
            └── *.md         # Themed pages

---

### Step 6: Initial Ingest — Project Analysis

**Task:** Study the existing project code and create initial wiki pages.

**What to do:**

1. **Read** all project source files (structure, dependencies, patterns)
2. **Create** wiki pages:

    | Page | Content |
    |------|---------|
    | `Architecture.md` | Overall architecture, components, tech stack |
    | `Components.md` | Description of each component |
    | `Database.md` | DB schema, tables, access layer |
    | `Code Patterns.md` | Patterns used, conventions, code style |
    | `Testing.md` | Test projects, testing patterns |

3. **Update** `wiki/index.md` — add all created pages to the catalog
4. **Update** `wiki/log.md` — add an entry about the initial ingest

**Rule:** Wiki contains **overview + links to files**, not full code copies.

**Example wiki page format:**

    ---
    tags: [component, description]
    created: YYYY-MM-DD
    updated: YYYY-MM-DD
    sources: [path/to/file1, path/to/file2]
    ---

    # Component Name

    ## Overview

    Brief description — what the component does, its role in the system.

    **Type:** Console App / Web API / Class Library / BackgroundService

    **Sources:** `path/to/component/`

    ---

    ## Structure

        component file tree

    ---

    ## Dependencies

    | Project/Package | Purpose |
    |-----------------|---------|
    | Dependency 1 | What it's used for |

    ---

    ## Data Flow

    1. Step 1
    2. Step 2
    3. Step 3

---

## FINAL RESULT

After completing all steps, the project should have:

    project/
    ├── QWEN.md                        # Main rules (wiki-first)
    ├── docs/
    │   └── wiki/
    │       ├── index.md               # Wiki page catalog
    │       ├── log.md                 # Change log
    │       ├── Wiki Format.md         # Format guide
    │       ├── Architecture.md        # Architecture overview
    │       ├── Components.md          # Component descriptions
    │       └── ...                    # Other pages
    └── .qwen/
        └── skills/
            ├── wiki-workflow/
            │   └── SKILL.md           # Wiki management skill
            ├── code-contributor/
            │   └── SKILL.md           # Code writing skill
            └── code-architecture/
                └── SKILL.md           # Architecture review skill

---

## VERIFICATION CHECKLIST

After creating all artefacts, verify:

- [ ] `QWEN.md` contains the wiki-first rule
- [ ] `docs/wiki/index.md` exists and contains the catalog
- [ ] `docs/wiki/log.md` exists with an initial entry
- [ ] `docs/wiki/Wiki Format.md` exists with the format guide
- [ ] All 3 skills in `.qwen/skills/` are created
- [ ] Each skill contains the wiki-first rule at the top
- [ ] Source priority: wiki > code
- [ ] Initial wiki pages are created (architecture, components, patterns)
- [ ] `index.md` is updated with links to all pages
- [ ] `log.md` contains an entry about initial page creation

---

## AFTER SETUP — HOW TO WORK

### For developers

1. Before a task → open `docs/wiki/index.md`, find the relevant page
2. After changing code → update wiki pages + add an entry to `log.md`

### For the AI agent

1. For any task → wiki first, then code
2. After code changes → post-change lint (update wiki)
3. For "how does X work?" → query wiki

### Typical workflow

    Task: "Add a new endpoint"
      1. Wiki-first: read wiki/pages about API
      2. Read existing controller code
      3. Write code following wiki patterns
      4. Update wiki: add new endpoint description
      5. Add entry to log.md
