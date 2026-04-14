# AI Assistants — The Wiki-First Approach

In modern projects, the codebase grows exponentially, and AI assistants have become an integral part of the development workflow. However, without a single source of truth, every developer (and every AI agent) is forced to learn the project from scratch, wasting context window limits, time, and resources. The **wiki-first** approach solves this problem by turning documentation into a living, automatically maintained knowledge center that dictates how both humans and AI operate.

> 📌 **Note:** All structural examples, file names (`QWEN.md`, `.qwen/skills/`), and workflows in this article are provided for the **QWEN** ecosystem. The methodology itself is universal and can easily be adapted to any AI agent capable of working with Markdown instructions.

---

## 🔍 What is Wiki-First?

**Wiki-first** is a methodology where, before any work with the code (writing features, reviewing, refactoring, making architectural decisions), the developer or AI agent **must** first study the project's wiki documentation (`docs/wiki/`).

Key principle:
> `Wiki = overview, Code = details`

The wiki stores condensed, structured knowledge: system architecture, accepted patterns, service dependencies, naming conventions, and DI rules. Specific method signatures, SQL queries, or test assertions are looked up directly in the code if the wiki does not cover the task.

---

## 🚀 Benefits of the Approach

| Aspect | Without Wiki-First | With Wiki-First |
|--------|----------------|--------------|
| **Project Onboarding** | Everyone learns the code from scratch | Ready-made overview in 5 minutes |
| **Knowledge Storage** | Knowledge stays in developers' heads | Knowledge is centralized in the wiki, accessible to all |
| **Onboarding** | Newcomers spend days deciphering the code | Newcomers read the wiki → hours instead of days |
| **AI Agent Workflow** | Wastes context scanning thousands of lines | Reads concise wiki pages, saving tokens and time |
| **Documentation Freshness** | Wiki falls behind the code, quickly becomes outdated | Wiki is automatically updated after every change |
| **Standard Compliance** | Developers violate patterns out of ignorance | AI and humans follow unified rules from the wiki |

---

## 🛠 How to Set Up Wiki-First in Your Project

For quick and correct setup, use the [WIKI_FIRST_TEMPLATE.en.md](WIKI_FIRST_TEMPLATE.en.md) template. It is designed **for execution by an AI agent**, which will automatically create the entire knowledge infrastructure and skills.

### Step 1. Prepare Parameters
1. Copy `WIKI_FIRST_TEMPLATE.en.md` to the root of your repository.
2. Open the file and fill the `INPUT DATA` table with real values:

| Parameter | Example |
|----------|--------|
| `project_name` | `WebArchiveBackend` |
| `project_description` | `Backend API for storing and indexing web pages` |
| `tech_stack` | `C#, .NET 8, gRPC, PostgreSQL, Dapper` |
| `docs_path` | `docs/` |
| `wiki_path` | `docs/wiki/` |
| `skills_path` | `.qwen/skills/` |
| `source_dirs` | `src/, tests/` |

### Step 2. Launch the AI Agent
Pass the filled file to your AI assistant with the following prompt:
> "Execute the instructions from `WIKI_FIRST_TEMPLATE.en.md` strictly step-by-step. Create the wiki structure, AI skills, and the main rules file. Do not modify the existing project code. After creating the structure, perform the Initial Ingest (Step 6)."

The agent will automatically:
1. Create the `docs/wiki/` directory and three base files:
   - `index.md` — catalog of all wiki pages
   - `log.md` — change log (append-only)
   - `Wiki Format.md` — guide to page formatting
2. Generate three specialized AI skills in `.qwen/skills/`:
   - `wiki-workflow` — knowledge management (ingest, query, lint, post-change lint)
   - `code-contributor` — writing and modifying code according to standards
   - `code-architecture` — architectural review (SOLID, DI, layered architecture)
3. Create `QWEN.md` in the project root with a strict wiki-first rule and source hierarchy: `docs/wiki/ > Project code`.

### Step 3. Initial Project Analysis (Initial Ingest)
After creating the structure, the agent will execute **Step 6** from the template:
- Scans the source code directories (`source_dirs`).
- Generates initial wiki pages: `Architecture.md`, `Components.md`, `Database.md`, `Code Patterns.md`, `Testing.md`.
- Updates `index.md`, adding new pages to the catalog.
- Adds an entry to `log.md` about the initial knowledge load.
- ⚠️ **Important:** The wiki contains only overviews and relative links to files. Copying code from `.cs`, `.proto`, or `.sql` files is prohibited.

### Step 4. Verify Installation
After the agent finishes, verify the result using the checklist:
- [ ] `QWEN.md` contains the wiki-first rule and source priority
- [ ] `docs/wiki/` contains `index.md`, `log.md`, `Wiki Format.md`
- [ ] `.qwen/skills/` contains 3 `SKILL.md` files
- [ ] All skills explicitly state: `wiki > code`
- [ ] Base wiki pages for architecture, components, and patterns are generated
- [ ] `index.md` contains links to all created pages
- [ ] `log.md` contains an entry about the first ingest with date and description

---

## 🔄 How to Work with Wiki-First After Setup

| Role | Rule |
|------|---------|
| **Developer** | Before a task → open `docs/wiki/index.md`. After committing → update wiki + add an entry to `log.md`. |
| **AI Agent** | Any request → wiki first. After code changes → post-change lint. Missing details → read code. |
| **Code Review** | If a PR didn't update the wiki → request changes. Wiki must not diverge from the code. |

Typical AI agent workflow:
1. `Wiki-first` → reads `index.md` → finds relevant pages
2. `Query/Ingest` → clarifies details in code if needed
3. `Code` → writes feature/fix according to wiki patterns
4. `Post-Change Lint` → updates wiki and `log.md`
5. `Verify` → checks links, frontmatter, code alignment

---

## 🔄 Wiki Lifecycle: Maintenance, Consistency, and Skill Evolution

Setting up wiki-first is not a one-time configuration, but the launch of a living process. Documentation left unattended quickly becomes a "knowledge graveyard." To ensure the approach delivers long-term value, the wiki must be continuously updated, and the AI agent must act as the primary guarantor of its consistency and completeness.

### 🤖 Agent as the Guardian of Integrity
After the initial setup, responsibility for wiki freshness falls on the AI agent. With every code change, the agent must automatically run a **post-change lint**: check if existing pages contradict the new implementation, update descriptions of affected components, and log changes in `log.md`.

Require explicit synchronization confirmation from the agent. If a commit affects architecture, configuration, or adds new modules, the agent must not only fix the code but also update the wiki. This turns documentation from an "extra burden" into an integral part of the development cycle. Without this step, the wiki will quickly drift from the code, and the `wiki > code` rule will lose its meaning.

### 📖 Documenting Complex Scenarios
The true power of wiki-first emerges when not only static descriptions but also **project-specific workflows** are added to the knowledge base.

For example, adding a new configuration parameter in your project might require changes in five different places: from `appsettings.json` to DI container objects, DTO models, and DB migrations. Instead of explaining this to the agent from scratch every time, document this scenario once as a separate wiki page or a section in `Code Patterns.md`.

Once a complex scenario is documented in the wiki, the agent begins executing it **confidently, without hallucinations or missed steps**. When asked "Add parameter X", the agent:
1. Finds the instruction in the wiki via `index.md`.
2. Strictly follows the described sequence of changes.
3. Automatically updates the wiki if the scenario needs adaptation to new realities.

The more "routine" and "easily forgotten" scenarios you formalize in the wiki, the less time is spent micromanaging the agent, and the higher the stability of the results.

### 🧠 Model-Specific Nature of Wiki
Different LLM models structure thoughts, choose priorities, and formulate instructions differently. This is **not a bug, but a feature**. When an agent itself generates the wiki during initial analysis (`Initial Ingest`) or subsequent updates, the documentation naturally "adapts" to its internal logic, tokenization, and attention patterns.

A model will follow a wiki it generated or curated itself much more accurately and confidently than documentation written by another model or a human "for themselves." Therefore, do not strive for perfect "human" stylistics in the early stages. The main thing is that the structure is unambiguous, links work, and the update logic is maintained. Over time, you will notice that the wiki becomes the "native language" of your agent: it finds context faster, hallucinates less, and executes complex tasks more precisely.

### 🛠 Evolution of Skills and Rules
The `skills` created during setup are not set in stone. As the project grows and experience with the agent accumulates, they must be adapted:
- If the agent frequently misses a certain type of check, add the corresponding rule to `code-architecture` or `wiki-workflow`.
- If new tech stacks or patterns emerge, expand `code-contributor` with examples and conventions.
- If the wiki structure becomes too bulky, refactor `index.md`, introduce tags, or split monolithic pages into thematic blocks.

The `QWEN.md` and `SKILL.md` files themselves are part of the wiki ecosystem. They can and should be refactored to keep instructions precise, minimize context consumption, and match the current maturity stage of the project.

### 💡 Core Principle: Wiki is a Living Organism
The key idea of the approach is this: **you cannot just create a wiki, you must continuously develop it**.
- **Maintain freshness:** every code change → wiki change + entry in `log.md`.
- **Enrich context:** turn verbal agreements, bug trackers, and complex manual procedures into formalized wiki instructions.
- **Evolve the agent:** adapt skills, clarify priorities, and refine prompts for real project tasks.

When the wiki becomes the single source of truth, and the agent acts as its active curator, the team gets a scalable knowledge system. It doesn't "wither" over time but gains strength: the more you invest in it, the faster, more accurate, and more confident the AI works, reducing time spent on onboarding, code reviews, and bug fixing.

---

## 💡 Conclusion

The wiki-first approach transforms documentation from an "artifact that gets forgotten" into an **active development tool**. AI agents operating under these rules save context, strictly adhere to architectural standards, and accelerate feature delivery.

Set up wiki-first once using [WIKI_FIRST_TEMPLATE.en.md](WIKI_FIRST_TEMPLATE.en.md), integrate it into your CI/CD and onboarding processes, and you will get a scalable knowledge system that grows with your project rather than turning into legacy.