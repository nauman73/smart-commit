# 📝 Smart Commit

An AI agent skill that stages changes and builds commit messages following your branch's established conventions — with full user review before anything is committed.

## 🚀 Quick Start

Invoke the skill in your AI coding tool:

```
/smart-commit
```

Or simply ask the agent to commit your changes — the skill triggers automatically when you mention committing.

## 🔄 Workflow

The skill walks through five sequential steps, pausing for your input at each stage:

| Step | What happens |
|------|-------------|
| **1. Gather context** | Reads `git status`, `git diff`, and recent commit history |
| **2. Stage files** | Stages modified tracked files, asks about untracked files, then confirms the final file list with you |
| **3. Build message** | Asks you to pick a commit message format, then assembles the first line and body |
| **4. Review** | Presents the full commit message for your approval or editing |
| **5. Commit** | Lets you choose **Commit** or **Commit and push**, then executes |

## 📋 Commit Message Conventions

The skill supports reusable commit message conventions stored in `references/conventions/`. Each convention defines a first-line format with named fields that the skill prompts you to fill in.

### Built-in Conventions

#### 🏷️ conventional-commits

The [Conventional Commits](https://www.conventionalcommits.org/) standard with optional scope.

- **Format:** `<type>(<scope>)?: <subject>`
- **Example:** `feat(auth): add JWT login`

### ➕ Creating Custom Conventions

When prompted for a commit message format, choose **"Define a new convention"**. The skill will interview you for:

1. A **name** for the convention
2. A **one-line description**
3. An **example first line** or template

The convention is saved to `references/conventions/<name>.md` and appears in the menu on future runs.

## 📂 Directory Structure

```
smart-commit/
├── 📄 SKILL.md                          # Skill definition and full prompt
├── 📄 README.md                         # This file
└── 📁 references/
    └── 📁 conventions/                  # Saved commit message conventions
        └── 📄 conventional-commits.md   # Conventional Commits format
        └── ...                          # Add your own conventions here
```

## 🛡️ Safety Features

- ⛔ Never uses `git add -A` or `git add .` — files are staged explicitly by name
- 🔒 Never stages files that may contain secrets (`.env`, credentials, keys)
- ✋ Never commits without user approval of the message
- 🚫 Never pushes to remote without asking first
- 🔁 Never amends a previous commit unless explicitly asked
- ✅ Never skips pre-commit hooks (`--no-verify`)

## 💡 Tips

- Type **`a`** when asked about the commit format to reuse the most recent commit's first line — great for follow-up commits in the same area.
- Conventions persist across sessions, so you only define a format once.
- If you have no files to commit, the skill will show you what's available and let you choose before stopping.
