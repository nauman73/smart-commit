---
name: smart-commit
description: Commit staged and unstaged changes following the branch's established commit message conventions. Use this skill whenever the user asks to commit, wants to commit changes, says "/smart-commit", or mentions committing recent work. Handles staging, message generation, user review, and the final commit.
---

# Smart Commit

Commit all changes since the last commit, following the branch's established first-line format and presenting the message for user review before committing.

## Skill directory

When this skill is triggered, the system message includes a line like `Base directory for this skill: /path/to/smart-commit`. That path is referred to as `SKILL_DIR` throughout these instructions. All references to `references/conventions/` below mean `SKILL_DIR/references/conventions/` — **not** the user's current working directory. Always resolve paths relative to the skill's base directory.

## Flags

- `--fresh` — Skip preference recall (Step 0) and run through all steps from scratch, prompting at each.

## Workflow

**Execute steps strictly in order.** Complete each step fully — including any user confirmations — before moving to the next. Never combine questions or decisions from multiple steps into a single prompt. For example, do not ask about the commit message format (Step 3) while still confirming the file list (Step 2).

### Step 0: Recall previous preferences

This step provides a shortcut for repeat commits within the same session. If the `--fresh` flag was passed, skip this step entirely and go to Step 1.

Look back through the conversation history for a previous invocation of this skill in the current session. If none is found, skip to Step 1.

If a previous invocation exists, extract the preferences that were used:

- **Staging approach** — whether untracked files were included or excluded.
- **Convention + field values** — which convention was used and the values supplied for each field (e.g., `type` = feat, `scope` = auth).
- **Action** — commit only or commit and push.

Run `git status` and `git diff` to determine what files currently have changes. Then present the recalled preferences along with the current file list (derived by applying the recalled staging approach to the current status):

> **Previous smart-commit preferences found:**
> - Staging: tracked files only (untracked files were excluded)
>   - Files to commit: `src/auth.ts`, `src/login.ts`, `tests/auth.test.ts`
> - Convention: `conventional-commits` (`type` = feat, `scope` = auth)
> - Action: commit and push
>
> Use same preferences? (Y/N)

**If the user says Yes:**

Execute Steps 1–5 in fast mode — skip all user prompts except for the message review in Step 4 (since the body is freshly generated from the new diff). Specifically:

- **Step 1** — Gather context as normal.
- **Step 2** — Auto-stage files per the recalled staging approach. Show the file list for awareness but do not prompt for confirmation.
- **Step 3** — Build the first line using the recalled convention and field values (no field interview). Generate the body fresh from the current diff.
- **Step 4** — Present the full commit message for review. The user must still confirm here because the body is new.
- **Step 5** — Execute the recalled action (commit or commit-and-push) without asking again.

**If the user says No:**

Proceed to Step 1 and run through all steps normally, prompting at each.

### Step 1: Gather context

Run these sequentially in a single command (parallel bash calls can fail on some environments):

```bash
git status
git diff
git diff --cached
git log --oneline -5
```

If there are no changes (no modified, no staged files), tell the user there's nothing to commit and stop.

### Step 2: Stage files

Stage all **modified tracked files** — the files that show up under "Changes not staged for commit" in `git status`. Use `git add <file1> <file2> ...` with explicit file names.

**Untracked files** require judgment based on the current conversation context:

- If there are untracked files that appear related to work done in this chat session (e.g., files you created or that the user asked you to create), list them and ask the user whether they should be staged. Present the list clearly so the user can confirm or decline.
- If there is no session context to determine whether untracked files are related to the current work, inform the user that untracked files exist but will be ignored since they cannot be attributed to the current session's changes.

Never stage files that likely contain secrets (`.env`, credentials, keys) — warn the user if they ask to include such files.

**Ensure there are files to commit.** After processing tracked and untracked files above, check whether there are any files staged for commit (either already staged from Step 1, or newly staged in this step). If the combined set is empty:

1. Present the user with the full picture from `git status` — list any untracked files, unstaged changes, or other details that might help them decide what to include.
2. Ask the user which files, if any, should be staged.
3. If the user declines to stage anything (or there are genuinely no files available), stop and tell the user: "There are no files to commit. Stopping." Do not proceed to Step 3.

**Confirm the final file list.** Once there is at least one file to commit, present the complete list of files that will be included in the commit (both previously staged and newly staged) and ask the user to confirm before moving on. If the user wants to add or remove files, adjust accordingly and re-confirm.

### Step 3: Build the commit message

The commit message has two parts:

**First line:**

The first line can come from three sources: the most recent commit, a saved convention, or a new convention the user defines on the spot. Saved conventions live in `SKILL_DIR/references/conventions/` as one Markdown file per convention (YAML frontmatter with `name` + `description`, then `## Format`, `## Example`, `## Fields` sections). They persist across sessions so the user doesn't re-describe their format every run.

1. **Gather the options:**
   - Read the most recent commit first line via `git log --oneline -1` (strip the leading short hash).
   - List files in `SKILL_DIR/references/conventions/*.md`. For each, read the frontmatter to get `name` and `description`.

2. **Ask the user which source to use.** Adapt the question to what's available:
   - **If saved conventions exist**, present three choices:
     - (a) Use the most recent commit's first line (show it).
     - (b) Use a saved convention — list each as `<name> — <description>`, numbered for easy selection.
     - (c) Define a new convention.
   - **If no saved conventions exist yet**, present two choices: (a) use most recent, or (b) define a new convention.
   - **If there is no prior commit on the branch** (fresh repo), skip option (a).

3. **If the user picks "use most recent"** → use that line verbatim as the first line.

4. **If the user picks a saved convention:**
   - Read the full convention file.
   - For each field listed under `## Fields`, ask the user for a value. Respect any per-field rules noted in the convention (e.g., auto-prefix behaviour).
   - Assemble the first line by substituting the values into the pattern shown in `## Format`.

5. **If the user picks "define a new convention":**
   a. Ask for a **name** (required). Slugify it for the filename (lowercase, spaces/punctuation → hyphens). If `SKILL_DIR/references/conventions/<slug>.md` already exists, tell the user and ask for a different name.
   b. Ask for a **one-line description** — this is what future runs show in the menu, so it should be distinctive.
   c. Ask for an **example first line** (or a template with angle-bracket placeholders like `<version>`). From this, infer the `## Format` pattern and the list of `## Fields`.
   d. Echo back the inferred `## Format`, `## Example`, and `## Fields` and ask the user to confirm or correct. This prevents saving a misunderstood format.
   e. Once confirmed, write `SKILL_DIR/references/conventions/<slug>.md` using the frontmatter + sections layout described above.
   f. Then fall through to step 4's field interview to build the first line for this commit.

**Body (after a blank line):** Write a concise summary of what changed, focusing on the "why" rather than listing every file. Keep it to 2-5 sentences. Mention key behavioral changes, new features, or bug fixes.

### Step 4: Present for review

Show the full commit message to the user and ask them to approve or request changes. Do NOT commit until the user confirms.

Format it clearly so the user can read it:

```
Proposed commit message:

CRMC 5.2.0 / CRMFD-7495 - Installer changes

<body text here>
```

Then ask: "Does this look good?"

- If the user says **Yes** (or **Y**), proceed to Step 5.
- If the user says **No** (or **N**), ask what changes they'd like, apply them, and present the updated message for another round of review.

### Step 5: Commit

Once the commit message is approved, ask the user to choose:

1. **Commit** — commit only
2. **Commit and push** — commit and push to remote

Then create the commit. Pass the message as a multi-line double-quoted string — this works reliably across all platforms including Windows Git Bash, where HEREDOC syntax (`$(cat <<'EOF' ... EOF)`) crashes the shell process.

```bash
git commit -m "First line of commit message

Body text here."
```

If the commit message body contains double quotes, escape them with `\"`.

After committing, run `git status` to confirm success and report the commit hash to the user.

If the user chose **Commit and push**, run `git push` (or `git push -u origin <branch>` if the branch has no upstream set).

If a pre-commit hook fails, investigate the issue, fix it, re-stage, and create a NEW commit (do not amend).

## Important

- Never push to remote without asking the user first.
- Never amend a previous commit unless the user explicitly asks.
- Never use `git add -A` or `git add .`.
- Never skip hooks (`--no-verify`).
- If the user requests changes to the message, apply them and show the updated message for another round of review.
