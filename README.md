# 🚀 AI Agent Skills

A collection of production-grade skills, prompts, and architectural runbooks for AI coding agents and assistants (**Google Antigravity**, **Claude Code**, **Cursor**, **GitHub Copilot**, **Windsurf**, **Cline / Roo Code**, and more).

Author: **Adesina Mark Omoniyi**

📖 **[Read the 4-Phase Application Building Workflow Guide](./APPLICATION-WORKFLOW.md)** to see how these skills connect end-to-end.

---

## 📚 Skills Catalog

| Skill | Version | Description | Tags |
| :--- | :--- | :--- | :--- |
| [**`aspnet-core-scaffolding`**](./aspnet-core-scaffolding/) | `1.0.0` | Enterprise ASP.NET Core standards: Clean Architecture, CQRS, Dapper, Dispatcher, FluentValidation, and Result Pattern. | `dotnet`, `aspnetcore`, `clean-architecture`, `cqrs`, `dapper` |
| [**`add-feature`**](./add-feature/) | `1.0.0` | Interactive vertical slice feature scaffolding with database table, schema, and domain requirement discovery. | `dotnet`, `aspnetcore`, `clean-architecture`, `cqrs`, `feature-scaffolding` |
| [**`architect`**](./architect/) | `1.0.0` | Senior engineering architectural thinking partner: domain vocabulary alignment, trade-offs, and implementation blueprints. | `software-architecture`, `system-design`, `senior-engineer`, `planning` |
| [**`explain-this`**](./explain-this/) | `1.0.0` | Structured deep-dive code analysis, execution flow breakdowns, architectural reviews, and edge-case dissection. | `code-explanation`, `code-analysis`, `dotnet`, `csharp` |

---

## 📦 Multi-Platform Installation Guide

Agent skills are markdown-based execution guides (`SKILL.md`) that teach AI agents specific workflows, design patterns, and coding standards. Choose your tool below to install and activate them:

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

Antigravity natively discovers skills placed in your global config or workspace root.

#### Option A: Global Installation (Available across all workspaces)

- **Windows (PowerShell):**
  ```powershell
  # 1. Create global skills directory if it doesn't exist
  New-Item -ItemType Directory -Force -Path "$HOME\.gemini\config\skills"

  # 2. Clone repository into global skills
  git clone https://github.com/developerslearnit/skills.git "$HOME\.gemini\config\skills\agent-skills"
  ```

  > **Note:** If you only want to install a specific skill (e.g. `aspnet-core-scaffolding`):
  > ```powershell
  > Copy-Item -Recurse -Force "aspnet-core-scaffolding" "$HOME\.gemini\config\skills\aspnet-core-scaffolding"
  > ```

- **macOS / Linux (Bash):**
  ```bash
  # 1. Create global skills directory
  mkdir -p ~/.gemini/config/skills

  # 2. Clone repository
  git clone https://github.com/developerslearnit/skills.git ~/.gemini/config/skills/agent-skills
  ```

#### Option B: Project-Level Installation (Workspace scoped)
Place the skill inside `.agents/skills/` at your project root:

- **Using Git Submodule:**
  ```bash
  git submodule add https://github.com/developerslearnit/skills.git .agents/skills/agent-skills
  ```

- **Or Direct Copy:**
  ```text
  your-project-root/
  └── .agents/
      └── skills/
          ├── aspnet-core-scaffolding/
          │   └── SKILL.md
          ├── add-feature/
          │   └── SKILL.md
          ├── architect/
          │   └── SKILL.md
          └── explain-this/
              └── SKILL.md
  ```

---

### 2. 🧠 Claude Code & Claude Desktop (Anthropic)

#### Claude Code (CLI)
Claude Code automatically reads instructions from `CLAUDE.md` in your project root or user home directory.

- **Option A: Link or Include in Project `CLAUDE.md`**
  Add references to the skill files directly into your project's `CLAUDE.md`:
  ```markdown
  # Project Guidelines & Agent Skills

  When scaffolding ASP.NET Core endpoints, adding features, architecting solutions, or explaining complex logic, follow the instructions in:
  - `.agents/skills/aspnet-core-scaffolding/SKILL.md`
  - `.agents/skills/add-feature/SKILL.md`
  - `.agents/skills/architect/SKILL.md`
  - `.agents/skills/explain-this/SKILL.md`
  ```

- **Option B: Add as Slash Commands / Custom Prompts**
  Copy the skill instructions to `.claude/commands/` or invoke Claude Code with the skill context:
  ```bash
  claude --prompt "Read .agents/skills/aspnet-core-scaffolding/SKILL.md and scaffold a new CreateUser command"
  ```

- **Option C: Global Claude Memory**
  Add the skill markdown content or references into `~/.claude/CLAUDE.md`.

#### Claude Desktop / Claude Web Projects
1. Open your **Project** in Claude Web or Desktop.
2. Under **Project Knowledge**, upload the `SKILL.md` files (e.g. `aspnet-core-scaffolding/SKILL.md`).
3. Under **Custom Instructions**, add:
   > *"When scaffolding code or explaining architecture, strictly adhere to the standards provided in the uploaded SKILL.md files."*

---

### 3. ⚡ Cursor AI

Cursor supports `.cursor/rules/` (MDC rule files) and `.cursorrules` in your project root.

#### Option A: Modern MDC Rules (`.cursor/rules/`)

Create `.cursor/rules/` in your project root and copy the skills with the `.mdc` extension:

- **Windows (PowerShell):**
  ```powershell
  New-Item -ItemType Directory -Force -Path ".cursor\rules"
  Copy-Item -Path "aspnet-core-scaffolding\SKILL.md" -Destination ".cursor\rules\aspnet-core-scaffolding.mdc"
  Copy-Item -Path "add-feature\SKILL.md" -Destination ".cursor\rules\add-feature.mdc"
  Copy-Item -Path "architect\SKILL.md" -Destination ".cursor\rules\architect.mdc"
  Copy-Item -Path "explain-this\SKILL.md" -Destination ".cursor\rules\explain-this.mdc"
  ```

- **macOS / Linux (Bash):**
  ```bash
  mkdir -p .cursor/rules
  cp aspnet-core-scaffolding/SKILL.md .cursor/rules/aspnet-core-scaffolding.mdc
  cp add-feature/SKILL.md .cursor/rules/add-feature.mdc
  cp architect/SKILL.md .cursor/rules/architect.mdc
  cp explain-this/SKILL.md .cursor/rules/explain-this.mdc
  ```

*Add frontmatter to the `.mdc` file to configure activation triggers (e.g. `globs: **/*.cs` or `alwaysApply: true`).*

#### Option B: Project `.cursorrules`
Append the contents of the relevant `SKILL.md` directly into the `.cursorrules` file at the root of your project:
```bash
cat aspnet-core-scaffolding/SKILL.md >> .cursorrules
```

#### Option C: Global Cursor Rules
In Cursor, navigate to **Settings > Cursor Settings > General > Rules for AI**, and paste the instructions from `SKILL.md`.

---

### 4. 🤖 GitHub Copilot (VS Code, Visual Studio & JetBrains)

GitHub Copilot supports custom repository instructions via `.github/copilot-instructions.md`.

#### Project-Level Setup (`.github/copilot-instructions.md`)

- **Windows (PowerShell):**
  ```powershell
  New-Item -ItemType Directory -Force -Path ".github"
  Get-Content "aspnet-core-scaffolding\SKILL.md" | Out-File -Append -FilePath ".github\copilot-instructions.md"
  ```

- **macOS / Linux (Bash):**
  ```bash
  mkdir -p .github
  cat aspnet-core-scaffolding/SKILL.md >> .github/copilot-instructions.md
  ```

#### VS Code Workspace Settings (`.vscode/settings.json`)
You can configure Copilot Chat custom instructions in your `.vscode/settings.json`:
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

Windsurf supports workspace rules through `.windsurfrules` or `.windsurf/rules/`.

#### Option A: Workspace Rules (`.windsurfrules`)
Copy or append the skill to `.windsurfrules` in your repository root:
```bash
cat aspnet-core-scaffolding/SKILL.md >> .windsurfrules
```

#### Option B: Global Memories / Rules
Add the skill to your global Windsurf configuration:
- **Windows:** `%USERPROFILE%\.codeium\windsurf\memories\global_rules.md`
- **macOS / Linux:** `~/.codeium/windsurf/memories/global_rules.md`

---

### 6. 🛠️ Cline / Roo Code (VS Code Extension)

Cline and Roo Code load instructions from `.clinerules` or `.roomodes`.

#### Option A: Workspace `.clinerules`
Copy the skill content into `.clinerules` in your project root:
```bash
cat aspnet-core-scaffolding/SKILL.md > .clinerules
```

#### Option B: Custom Modes (`.roomodes`)
Define custom modes (e.g., `Scaffolder` or `Architect`) in `.roomodes` referencing the skill instructions.

---

### 7. 💻 Aider & Continue.dev

#### Aider
Use Aider's `--read` flag or add the skill to your `.aider.conf.yml`:
```yaml
# .aider.conf.yml
read:
  - .agents/skills/aspnet-core-scaffolding/SKILL.md
```
Or run directly from the command line:
```bash
aider --read .agents/skills/aspnet-core-scaffolding/SKILL.md
```

#### Continue.dev
Add the skill file or system message in `.continue/config.json`:
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

- **For ASP.NET Core Scaffolding**:
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
├── aspnet-core-scaffolding/
│   ├── SKILL.md                 # Core instructions and scaffolding conventions
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
