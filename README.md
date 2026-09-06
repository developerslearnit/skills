# 🚀 AI Agent Skills

A collection of production-grade skills, prompts, and architectural runbooks for AI coding agents and assistants (**Google Antigravity**, **Claude Code**, **Cursor**, **GitHub Copilot**, **Windsurf**, **Cline / Roo Code**, and more).

Author: **Adesina Mark Omoniyi**

📖 **[Read the 4-Phase Application Building Workflow Guide](./APPLICATION-WORKFLOW.md)** to see how these skills connect end-to-end.

---

## 📚 Skills Catalog

| Skill | Version | Description | Tags |
| :--- | :--- | :--- | :--- |
| [**`aspnet-core-scaffolding`**](./aspnet-core-scaffolding/) | `1.0.0` | Enterprise ASP.NET Core standards: Clean Architecture, CQRS, Dapper, Dispatcher, FluentValidation, and Result Pattern. | `dotnet`, `aspnetcore`, `clean-architecture`, `cqrs`, `dapper` |
| [**`scaffold-aspnet-microservices`**](./scaffold-aspnet-microservices/) | `1.0.0` | Microservices architecture: Clean Architecture per service, SharedKernel with IDispatcher & CQRS, YARP Gateway, and Docker. | `dotnet`, `microservices`, `yarp`, `docker`, `sharedkernel`, `cqrs` |
| [**`add-feature`**](./add-feature/) | `1.0.0` | Interactive vertical slice feature scaffolding with database table, schema, and domain requirement discovery. | `dotnet`, `aspnetcore`, `clean-architecture`, `cqrs`, `feature-scaffolding` |
| [**`architect`**](./architect/) | `1.0.0` | Senior engineering architectural thinking partner: domain vocabulary alignment, trade-offs, and implementation blueprints. | `software-architecture`, `system-design`, `senior-engineer`, `planning` |
| [**`explain-this`**](./explain-this/) | `1.0.0` | Structured deep-dive code analysis, execution flow breakdowns, architectural reviews, and edge-case dissection. | `code-explanation`, `code-analysis`, `dotnet`, `csharp` |

---

## 📦 Installation via Skills CLI (`npx skills add`)

The fastest way to install these skills is via the official **[Skills CLI](https://skills.sh)** (`npx skills add`). It automatically fetches the skills from GitHub and configures them for your AI coding assistant:

```bash
# ⚡ Install all skills into your project
npx skills add developerslearnit/skills

# 🎯 Install a specific skill
npx skills add developerslearnit/skills --skill scaffold-aspnet-microservices
npx skills add developerslearnit/skills --skill aspnet-core-scaffolding
npx skills add developerslearnit/skills --skill add-feature
npx skills add developerslearnit/skills --skill architect
npx skills add developerslearnit/skills --skill explain-this

# 🌐 Install globally across all your projects
npx skills add developerslearnit/skills -g
```

---

## 🤖 Assistant-Specific Setup

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AI CODING ASSISTANTS                              │
├───────────────────┬───────────────────┬───────────────────┬─────────────────┤
│ Google Antigravity│    Claude Code    │     Cursor AI     │ GitHub Copilot  │
│  & Gemini Assist  │  & Claude Desktop │   (.cursor/rules) │ (.github/copilot│
├───────────────────┼───────────────────┼───────────────────┼─────────────────┤
│ Windsurf (Cascade)│  Cline / Roo Code │       Aider       │  Continue.dev   │
└───────────────────┴───────────────────┴───────────────────┴─────────────────┘
```

---

### 1. 🪐 Google Antigravity & Gemini Code Assist

Antigravity natively discovers skills placed in your workspace root (`.agents/skills/`) or global config (`~/.gemini/config/skills/`).

#### Option A: Workspace Installation (Recommended)
```bash
# Installs skills into your current project's .agents/skills/ directory
npx skills add developerslearnit/skills
```

#### Option B: Global Installation (Available across all workspaces)
```bash
# Installs skills into your global agent configuration
npx skills add developerslearnit/skills -g
```

---

### 2. 🧠 Claude Code & Claude Desktop (Anthropic)

#### Claude Code (CLI)
```bash
# Install skills for Claude Code
npx skills add developerslearnit/skills --agent claude-code
```

Alternatively, add references directly to your project's `CLAUDE.md`:
```markdown
# Project Guidelines & Agent Skills

When scaffolding ASP.NET Core endpoints, adding features, architecting solutions, or explaining complex logic, follow the instructions in:
- `.agents/skills/aspnet-core-scaffolding/SKILL.md`
- `.agents/skills/add-feature/SKILL.md`
- `.agents/skills/architect/SKILL.md`
- `.agents/skills/explain-this/SKILL.md`
```

#### Claude Desktop / Claude Web Projects
1. Open your **Project** in Claude Web or Desktop.
2. Under **Project Knowledge**, upload the `SKILL.md` files.
3. Under **Custom Instructions**, add:
   > *"When scaffolding code or explaining architecture, strictly adhere to the standards provided in the uploaded SKILL.md files."*

---

### 3. ⚡ Cursor AI

Cursor supports skills via `.cursor/rules/` (MDC rule files) and `.cursorrules`.

#### Install for Cursor
```bash
npx skills add developerslearnit/skills --agent cursor
```

Or reference the installed skills in your project's `.cursorrules`:
```markdown
Read and adhere to the guidelines in:
- .agents/skills/aspnet-core-scaffolding/SKILL.md
- .agents/skills/add-feature/SKILL.md
- .agents/skills/architect/SKILL.md
- .agents/skills/explain-this/SKILL.md
```

---

### 4. 🤖 GitHub Copilot (VS Code, Visual Studio & JetBrains)

GitHub Copilot supports repository instructions via `.github/copilot-instructions.md` and `.vscode/settings.json`.

#### Install for GitHub Copilot
```bash
npx skills add developerslearnit/skills --agent github-copilot
```

#### VS Code Workspace Settings (`.vscode/settings.json`)
```json
{
  "github.copilot.chat.codeGeneration.instructions": [
    {
      "file": ".agents/skills/aspnet-core-scaffolding/SKILL.md"
    },
    {
      "file": ".agents/skills/add-feature/SKILL.md"
    },
    {
      "file": ".agents/skills/architect/SKILL.md"
    },
    {
      "file": ".agents/skills/explain-this/SKILL.md"
    }
  ]
}
```

#### In-Chat Reference
You can also tag skills directly in Copilot Chat:
> *"@workspace #file:.agents/skills/aspnet-core-scaffolding/SKILL.md scaffold a new GetOrderByIdQuery handler"*

---

### 5. 🌊 Windsurf / Cascade (Codeium)

Windsurf supports workspace rules through `.windsurfrules`.

#### Install for Windsurf
```bash
npx skills add developerslearnit/skills --agent windsurf
```

Or append the instructions to `.windsurfrules` in your repository root.

---

### 6. 🛠️ Cline / Roo Code (VS Code Extension)

#### Install for Cline / Roo Code
```bash
npx skills add developerslearnit/skills --agent cline
```

---

### 7. 💻 Aider & Continue.dev

#### Aider
Add the skill to your `.aider.conf.yml`:
```yaml
read:
  - .agents/skills/aspnet-core-scaffolding/SKILL.md
```

#### Continue.dev
Add the skill file in `.continue/config.json`:
```json
{
  "customCommands": [
    {
      "name": "scaffold-aspnet",
      "prompt": "Read the guidelines in .agents/skills/aspnet-core-scaffolding/SKILL.md and generate the following CQRS handler: {{{ input }}}",
      "description": "Scaffold ASP.NET Core CQRS component"
    }
  ]
}
```

---

## 🛠️ How to Use

Once installed, AI assistants will adhere to these skill guidelines. You can trigger them naturally with prompts:

### Example Prompts:

- **For ASP.NET Core Microservices Scaffolding**:
  > *"Scaffold a new ASP.NET Core microservices solution called SupportlyAI with Tickets, Customers, and Knowledge services, YARP API Gateway, SharedKernel, and Docker Compose."*
  > *"Add a new microservice called IdentityService following the scaffold-aspnet-microservices guidelines."*

- **For Single-Service / Modular Monolith Scaffolding**:
  > *"Scaffold a new CreateSupportTicket command handler using CQRS and the Result pattern."*
  > *"Create a repository and query handler for getting tickets with Dapper and pagination."*

- **For Adding Features Interactively**:
  > *"Add a new feature to manage customer subscriptions, ask me for table and validation details."*
  > *"Add a CancelOrder command and endpoint to the existing Orders feature slice."*

- **For Architectural Thinking & Planning**:
  > *"Act as an architect and help me design a multi-tenant payment reconciliation engine."*
  > *"Let's think through the architecture for a real-time event streaming pipeline before coding."*

- **For Code Explanation**:
  > *"Explain how this Dispatcher middleware handles pipeline execution."*
  > *"Walk me through the TicketRoutingEngine class and identify potential failure points."*

---

## 📂 Repository Structure

```text
skills/
├── scaffold-aspnet-microservices/
│   ├── SKILL.md                 # Microservices standards: Clean Arch, SharedKernel, YARP, Docker
│   ├── manifest.json            # Skill metadata and tags
│   └── references/              # SharedKernel, YARP gateway, Docker Compose & architecture guides
├── aspnet-core-scaffolding/
│   ├── SKILL.md                 # Core single-service Clean Arch & scaffolding conventions
│   ├── manifest.json            # Skill metadata and tags
│   └── references/              # Architectural patterns, CQRS, and Result pattern guides
├── add-feature/
│   ├── SKILL.md                 # Interactive feature addition workflow & conventions
│   ├── manifest.json            # Skill metadata and tags
│   └── references/              # Interactive questionnaires and feature scaffolding templates
├── architect/
│   ├── SKILL.md                 # Senior engineering thinking partner & blueprint guide
│   ├── manifest.json            # Skill metadata and tags
│   └── references/              # Decision framework & blueprint template
├── explain-this/
│   ├── SKILL.md                 # Step-by-step code explanation instructions
│   ├── manifest.json            # Skill metadata and tags
│   └── references/              # Analysis frameworks and explanation templates
├── .gitignore
└── README.md                    # Multi-platform installation & usage guide
```

---

## 🤝 Contributing

Contributions, bug fixes, and new skills are welcome!
1. Fork the repository
2. Create a feature branch (`git checkout -b skill/my-new-skill`)
3. Commit your changes (`git commit -m "feat: add my new skill"`)
4. Push to the branch (`git push origin skill/my-new-skill`)
5. Open a Pull Request

---

## 📄 License

This repository is licensed under the MIT License.
