# 🚀 Antigravity Agent Skills

A collection of production-grade skills and runbooks for AI coding agents (**Google Antigravity**, **Gemini Code Assist**, and compatible agentic frameworks).

Author: **Adesina Mark Omoniyi**

---

## 📚 Skills Catalog

| Skill | Version | Description | Tags |
| :--- | :--- | :--- | :--- |
| [**`aspnet-core-scaffolding`**](./aspnet-core-scaffolding/) | `1.0.0` | Enterprise ASP.NET Core standards: Clean Architecture, CQRS, Dapper, Dispatcher, FluentValidation, and Result Pattern. | `dotnet`, `aspnetcore`, `clean-architecture`, `cqrs`, `dapper` |
| [**`explain-this`**](./explain-this/) | `1.0.0` | Structured deep-dive code analysis, execution flow breakdowns, architectural reviews, and edge-case dissection. | `code-explanation`, `code-analysis`, `dotnet`, `csharp` |

---

## 📦 Installation

You can install these skills **Globally** (available across all your projects on your machine) or **Per-Project** (scoped to a specific workspace or team repo).

### Option 1: Global Installation (Recommended)

Installing globally makes these skills available across all your workspaces in Antigravity.

#### Windows (PowerShell):
```powershell
# 1. Create the global skills directory if it doesn't exist
New-Item -ItemType Directory -Force -Path "$HOME\.gemini\config\skills"

# 2. Clone the skills into your global config
git clone https://github.com/<your-username>/<repo-name>.git "$HOME\.gemini\config\skills\agent-skills"
```

> **Note:** If you only want to install a specific skill (e.g. `aspnet-core-scaffolding`):
> ```powershell
> Copy-Item -Recurse -Force "aspnet-core-scaffolding" "$HOME\.gemini\config\skills\aspnet-core-scaffolding"
> ```

#### macOS / Linux (Bash):
```bash
# 1. Create the global skills directory
mkdir -p ~/.gemini/config/skills

# 2. Clone the repository
git clone https://github.com/<your-username>/<repo-name>.git ~/.gemini/config/skills/agent-skills
```

---

### Option 2: Project-Level Installation (Team / Workspace)

To make a skill available to everyone working in a specific project repository, place it inside `.agents/skills/`:

#### Using Git Submodule:
```bash
git submodule add https://github.com/<your-username>/<repo-name>.git .agents/skills/agent-skills
```

#### Or Copy Directly:
```text
your-project-root/
└── .agents/
    └── skills/
        ├── aspnet-core-scaffolding/
        │   └── SKILL.md
        └── explain-this/
            └── SKILL.md
```

---

## 🛠️ How to Use

Once installed, the agent automatically detects skills via **Progressive Disclosure**. You don't need to manually paste prompts or copy/paste instructions.

Simply prompt the agent naturally:

### Example Prompts:

- **For ASP.NET Core Scaffolding**:
  > *"Scaffold a new CreateSupportTicket command handler using CQRS and the Result pattern."*
  > *"Create a repository and query handler for getting tickets with Dapper."*

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
├── explain-this/
│   ├── SKILL.md                 # Step-by-step code explanation instructions
│   ├── manifest.json            # Skill metadata and tags
│   └── references/              # Analysis frameworks and explanation templates
├── .gitignore
└── README.md
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
