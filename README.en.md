# Agent Skill Check

[简体中文](README.md) · English

Three read-only review skills for AI agents. Use them to review a skill’s logic and structure, audit the security of skills and agent configurations, or screen GitHub repositories for application and supply chain risks.

## Choose a skill

| Skill | When to use it | What it checks |
| --- | --- | --- |
| [`person-skill-review`](person-skill-review/SKILL.md) | Authoring a skill or adapting an external skill | Goals, triggers, inputs, decision rules, dependencies, permissions, failure handling, and acceptance criteria. |
| [`skill-security-check`](skill-security-check/SKILL.md) | Reviewing a third-party skill before enabling it, or auditing a skill, plugin, MCP configuration, or agent prompt | Prompt injection, credential access, data disclosure, automatic execution, malicious scripts, hidden outbound requests, and unverified dependencies. |
| [`github-repo-security-check`](github-repo-security-check/SKILL.md) | Evaluating an application, tool, or codebase from GitHub | Injection, data leaks, authentication and authorization, dependency tampering, installation and build steps, CI, and deployment configuration. |

If a code repository contains agent assets that are loaded, installed, or distributed with the project, run both security reviews against the same commit or local snapshot and combine their findings.

## How to use

1. Pick a skill and keep its directory intact, including `SKILL.md` and `references/`. If your agent supports `SKILL.md`, follow its instructions to add and invoke the skill. You can also provide the directory as review guidance.
2. Provide the review target. A completeness review can start from text or a directory. For a security review, prefer the complete directory or provide a GitHub link with the commit, branch, and path scope when available.
3. Read the report for evidence, version, coverage, and blind spots. Review fixes against the same scope and verify the actual version before release or installation.

### Example requests

- “Use `person-skill-review` to check this skill’s triggers, error handling, and acceptance criteria. Suggest the smallest changes that address the issues.”
- “Use `skill-security-check` to review this third-party skill directory. Identify data transfers, tool permissions, and anything that needs verification before installation.”
- “Use `github-repo-security-check` to review the specified commit in this GitHub repository. Focus on credentials, unauthorized access, and build dependencies.”

## Review limits

These skills perform static review and treat target code, installation commands, and scripts as material to inspect. Findings apply to the commit or snapshot and scope identified in the report. A single `SKILL.md`, unverifiable dependencies, and unexamined binaries leave gaps in coverage. Production configuration, actual network traffic, and platform permissions require separate verification.
