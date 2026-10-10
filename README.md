# occupify-ai-engineer — Occupify Agent Customization System

This directory houses the centralized **AI Agent Customization & Squad Orchestration System** for the **Occupify** ecosystem, powered by Google Antigravity.

---

## 1. What This Directory Is About

The `.agents/` workspace customization root equips AI pair programmers and subagents with domain-specific knowledge, architectural guidelines, and specialized roles.

Instead of generic, one-size-fits-all responses, agents operating in this workspace:
- **Adopt Specialized Personas**: Act as architects, core logic developers, UI/UX engineers, or QA testers.
- **Enforce Architectural Directives**: Adhere to Bulletproof React, Feature-Driven Design, Dumb Data Isolation, and Zero-Deprecation policies.
- **Operate in Decoupled Plugins**: Isolate domain tools and guidelines into modular packages (e.g., `frontend`, `backend`).

---

## 2. How to Use

You can interact with agents directly in your IDE chat using natural language, direct mentions, or role delegation.

### 🔹 Example: Talking to the Frontend Tech Lead (`react-tech-lead`)

The **Frontend Tech Lead** acts as the primary architect and squad orchestrator for the `occupify-web` application.

#### Option A: Direct Mention
Invoke the Tech Lead directly in your prompt:
```text
@react-tech-lead: Please review the domain boundaries for the new JobFilterToolbar feature in src/features/jobs/.
```

#### Option B: Roleplay & Architectural Consultation
```text
Acting as react-tech-lead, design the API error matrix and folder layout for the Profile Editing Modal.
```

#### Option C: Squad Orchestration (Test-Driven Development)
The Tech Lead can orchestrate the entire frontend subagent squad following the **TDD Inversion Workflow**:
```text
Tech Lead, please lead the implementation of the Job Filter feature:
1. Coordinate with react-e2e-tester to write failing test contracts first.
2. Delegate data logic & query hooks to react-core-dev.
3. Delegate responsive UI & token compliance to react-ui-dev.
4. Have react-static-auditor run the final quality audit before sign-off.
```

### 🔹 Other Available Personas in Frontend Plugin
- `@react-core-dev`: Headless data, Zod validation schemas, TanStack Query hooks, Zustand stores.
- `@react-ui-dev`: Accessible UI components, Tailwind CSS v4 styling, compound component patterns.
- `@react-e2e-tester`: Test-first contracts, Vitest unit tests, Playwright journeys, responsive checks.
- `@react-static-auditor`: Static analysis, linting, dumb data isolation enforcement, zero deprecation checks.
- `@product-researcher-ba`: Market & competitor research, PRD authoring, Gherkin acceptance criteria.

### 🔹 Available Personas in Backend Plugin
- `@spring-tech-lead`: Lead Backend Architect and Squad Orchestrator. Microservice boundaries, OpenAPI contracts, and TDD delegation.
- `@spring-core-dev`: Core domain logic, Spring Data JPA repositories, transactions, Redis caching, RabbitMQ producers.
- `@api-gateway-dev`: REST/WebFlux controllers, ResponseFactory, @Authenticated argument resolvers, Gateway filters.
- `@backend-qa-tester`: Test-Driven Development, slice tests (@WebMvcTest, @DataJpaTest), Mockito boundaries.
- `@backend-quality-auditor`: OWASP security auditing, BaseException hierarchy compliance, unified response gatekeeping.

### 🔹 Available Personas in DevOps Plugin
- `@devops-engineer`: Docker multi-stage builds, root Docker Compose orchestration, and GitHub Actions CI pipelines.

---

## 3. Directory Architecture

The system is structured as modular **Antigravity Customization Elements & Plugins**:

```text
.agents/
├── OCCUPIFY.md                  # High-signal product & domain briefing for AI agents
├── README.md                    # System documentation and developer onboarding (this file)
│
├── rules/                       # Global workspace rules
│   └── verification.md          # Verification guidelines before declaring tasks complete
│
├── agents/                      # Global cross-cutting personas
│   └── product-researcher-ba.md # Product Owner & Business Analyst
│
└── plugins/                     # Modular domain-packaged bundles
    ├── frontend/                # 🚀 Frontend Plugin (React 19 + Vite 8)
    │   ├── plugin.json          # Plugin manifest
    │   ├── skills/              # Specialized domain runbooks & knowledge
    │   │   ├── react/           # Bulletproof React architecture manual & configs
    │   │   └── deprecation-audit/# Automated deprecation scanner & migration skill
    │   ├── agents/              # Domain-specific frontend personas
    │   │   ├── react-tech-lead.md
    │   │   ├── react-core-dev.md
    │   │   ├── react-ui-dev.md
    │   │   ├── react-static-auditor.md
    │   │   └── react-e2e-tester.md
    │   └── rules/               # Frontend-specific rule sets
    │       └── .gitkeep
    │
    ├── backend/                 # 🚀 Backend Plugin (Spring Boot 3 + Java 25 Microservices)
    │   ├── plugin.json          # Plugin manifest
    │   ├── skills/              # Comprehensive backend architecture skill
    │   │   └── spring-boot/     # Master Spring Boot skill (Clean Architecture, ResponseFactory, Sequential Flyway, RabbitMQ, Slice Testing)
    │   ├── agents/              # Domain-specific backend personas
    │   │   ├── spring-tech-lead.md
    │   │   ├── spring-core-dev.md
    │   │   ├── api-gateway-dev.md
    │   │   ├── backend-qa-tester.md
    │   │   └── backend-quality-auditor.md
    │   └── rules/               # Backend coding standards
    │       └── .gitkeep
    │
    └── devops/                  # 🛠️ DevOps Plugin (Containers, Compose & CI/CD)
        ├── plugin.json          # Plugin manifest
        ├── skills/              # Comprehensive DevOps & infrastructure skill
        │   └── devops/          # Master DevOps skill (Layertools Docker, Compose Ports 5530-5534, GitHub Actions CI)
        ├── agents/              # Infrastructure personas
        │   └── devops-engineer.md
        └── rules/
            └── .gitkeep
```

---

## 4. Guide to Contribute

You can easily extend the agent system as the Occupify project grows:

### A. Adding a New Skill
1. Create a directory under `plugins/<plugin-name>/skills/<skill-name>/`.
2. Add a `SKILL.md` with YAML frontmatter:
   ```markdown
   ---
   name: your-skill-name
   description: Clearly state what this skill does and when the agent should use it.
   ---
   # Your Skill Title
   Step-by-step procedures, commands, or references.
   ```
3. *(Optional)* Add a `references/` folder for in-depth documentation.

### B. Adding a New Persona (Subagent)
1. Create a markdown file under `plugins/<plugin-name>/agents/<agent-name>.md`.
2. Define the agent manifest in the frontmatter:
   ```markdown
   ---
   name: your-agent-name
   description: Role description and primary responsibilities.
   model: pro
   subagent: true
   tools:
     - view_file
     - write_to_file
     - replace_file_content
     - run_command
     - grep_search
     - list_dir
   ---
   # Persona Role & Responsibilities
   ...
   ```

### C. Adding a New Rule
- Place `.md` rule files in `.agents/rules/` for workspace-wide policies or in `.agents/plugins/<plugin-name>/rules/` for domain-specific guidelines.

### D. Creating a New Plugin (e.g., DevOps / Mobile)
1. Create a directory `.agents/plugins/<plugin-name>/`.
2. Add a `plugin.json` manifest:
   ```json
   {
     "name": "plugin-name",
     "version": "1.0.0",
     "description": "Short description of the plugin"
   }
   ```
3. Populate `skills/`, `agents/`, or `rules/` as needed. Antigravity will automatically discover and ingest it upon startup.