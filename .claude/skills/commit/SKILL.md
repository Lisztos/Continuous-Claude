---
name: commit
description: Create git commits with user approval and no Claude attribution
---

# Commit Changes

You are tasked with creating git commits for the changes made during this session.

## Process:

1. **Analyze all changes:**
   - Run `git status` and `git diff` to see everything
   - Review conversation history to understand what was accomplished and why

2. **Group changes into logical commits:**
   - **Always prefer multiple small, focused commits** over one large commit
   - Group by logical unit of work, not by file type
   - Each commit should represent one coherent change that could stand alone
   - Grouping heuristics:
     - Bug fix + its test → one commit
     - New feature + its test → one commit
     - Refactor that enables a feature → separate commit before the feature
     - Config/dependency changes → separate commit
     - Documentation updates → separate commit
     - Unrelated fixes → separate commits
   - Only combine everything into one commit if all changes are truly part of a single atomic change

3. **Present your plan to the user:**
   - List each planned commit with its files and message
   - Format:
     ```
     Commit 1: <message>
       - file_a.py
       - file_b.py
     Commit 2: <message>
       - file_c.py
     ```
   - Ask: "I plan to create [N] commit(s). Shall I proceed?"

4. **Execute upon confirmation (in order):**
   - For each commit: `git add <specific files>` then `git commit`
   - Use imperative mood in messages, focus on *why* not *what*
   - Never use `git add -A` or `git add .`
   - Show final result with `git log --oneline -n [number of commits]`

5. **Generate reasoning (after each commit):**
   - Run: `bash "$CLAUDE_CC_DIR/.claude/scripts/generate-reasoning.sh" <commit-hash> "<commit-message>"`
   - This captures what was tried during development (build failures, fixes)
   - The reasoning file helps future sessions understand past decisions
   - Stored in `.git/claude/commits/<hash>/reasoning.md`

## Important:
- **NEVER add co-author information or Claude attribution**
- Commits should be authored solely by the user
- Do not include any "Generated with Claude" messages
- Do not add "Co-Authored-By" lines
- Write commit messages as if the user wrote them

## Remember:
- You have the full context of what was done in this session
- Group related changes together
- Keep commits focused and atomic when possible
- The user trusts your judgment - they asked you to commit