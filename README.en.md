# Agent Skill Check

[简体中文](README.md) · English

## Why I built this

A client tried a skill someone had shared with them. The AI sent spam to many of their customers, damaging those relationships. I used these review skills to help the client find and fix the problem.

**Open source and lots of stars don't guarantee safety. Before installing a skill or running a GitHub project, ask your AI for a read-only review.**

**A skill is a set of instructions for an AI assistant.** This repository has three sets of read-only review instructions for workflows you write, skills others share, and code from GitHub.

## Choose a skill

- **A skill you wrote** → [`person-skill-review`](person-skill-review/) to spot logic gaps and check what the skill is allowed to do.
- **A skill you downloaded** → [`skill-security-check`](skill-security-check/) to check for prompt injection, credential access, and data sent outside your workspace.
- **A GitHub project** → [`github-repo-security-check`](github-repo-security-check/) to look for tampered dependencies, hidden network calls, and risky build scripts.

| Your question | Pick this skill | What it checks |
| --- | --- | --- |
| Did I leave any steps out of my AI instructions? | [`person-skill-review`](person-skill-review/SKILL.md) | Goals, triggers, inputs, error handling, permissions, and how to tell when the work is done. |
| Can I use a skill or plugin someone shared with me? | [`skill-security-check`](skill-security-check/SKILL.md) | Password access, data transfers, dangerous commands, and instructions that push the AI past its allowed permissions. |
| Does this GitHub project have security problems? | [`github-repo-security-check`](github-repo-security-check/SKILL.md) | Data leaks, unauthorized access, injection, risky dependencies, and risks in installation and deployment. |

## Get started in 30 seconds

1. If you've downloaded this repository, pick a skill above and copy **its entire folder** somewhere your agent can read it. Keep `SKILL.md` and `references/` together. If you haven't downloaded it yet, start with the first step under “How to use” below.
2. Replace the two bracketed items in the message below and send it to your agent.

> Read `SKILL.md` and `references/` in [the folder I just copied]. Follow those rules to review [the file, folder, or GitHub link] without running or changing it. Tell me the most important issues and what to do next.

If you want the agent to read a public GitHub link, add “I authorize read-only access to this repository for this review.”

## How to use

### First time: get the review instructions

1. On this repository's GitHub page, click the green **Code** button and then **Download ZIP**. Open your Downloads folder, unzip the file, and open the extracted folder whose name starts with `agent-skill-check` (usually `agent-skill-check-main`). The three folders inside match the names in the table above. [GitHub download help](https://docs.github.com/en/repositories/working-with-files/using-files/downloading-source-code-archives)
2. Open an AI chat that can read files. Use the **+** or paperclip next to the message box to attach the **review files** listed below. Send: “These are the review instructions. Tell me which files you could read. I will send the material to check next.” If you cannot attach a file, open it and paste its full text into the chat.
3. After the AI lists the rules it could read, send the **file, folder, or link you want checked** in the next message. Copy the matching prompt and replace the bracketed text with your own information. For example, replace `[attached file name]` with `travel-checklist.md`. If both the review instructions and the material are named `SKILL.md`, send them in separate messages.

If your AI already has the skill installed, select it and start with the relevant steps below. Reviewing a private project, a whole local folder, or a GitHub link requires appropriate file or network access. If your chat cannot access something, ask it to review the material it received and list what it could not see. Keep passwords, API keys, and customer data out of chat messages.

### 1. Check a skill you wrote

**Review files**: [SKILL.md](person-skill-review/SKILL.md) and [completeness-checklist.md](person-skill-review/references/completeness-checklist.md).

1. Attach the skill you wrote. It might be one `SKILL.md` file or a folder with supporting files. If your AI can read local folders, give it the folder path. For a quick trial, paste the text of your skill.
2. Copy the prompt below. Fill in the file or folder name and describe what you wanted the skill to do in one sentence.
3. Send the message. If the report recommends fixing the skill first, start with its top three issues. After editing, send the new version and ask: “Please check the issues from the previous report one by one.”

**Example prompt to copy**

```text
Review the skill I wrote using person-skill-review.
Material to check: [attached file name, local folder path, or the text I paste below].
What I want it to do: [one sentence about the goal].
Read the material without editing files or running scripts. Tell me if anything is missing. Then check when the skill should start, what it needs, what it should do when something goes wrong, and how to tell when it is finished.
Give me a simple verdict and the three most important issues. For each one, tell me where it is, why it matters, how to fix it, and how to check the fix. Explain anything you cannot determine.
```

**Try it**: Paste a short rule for a “Travel checklist helper”: “When the user names a destination, make a packing list.” Set the goal to “help me pack for a trip.” See if the AI asks you to specify the trip length, the season, or what to do when those details are missing.

### 2. Check a skill or plugin someone gave you

**Review files**: [SKILL.md](skill-security-check/SKILL.md) and [threat-checklist.md](skill-security-check/references/threat-checklist.md).

1. For a public GitHub project, copy the repository or skill folder link from your browser before installing it. If you already downloaded it, give its folder path to an AI that can read local files and start the review from a workspace you trust. Provide the whole folder if possible. With only one `SKILL.md`, the AI can review only that text.
2. Copy the prompt below and replace `[target]` with the link or folder path. Keep the network permission sentence if you gave a public GitHub link and want the AI to read it. For a local review, you can remove that sentence.
3. Read the verdict at the top. If it says to stop or get a manual review, ask: “Which file do I need to provide or which issue should I fix first?” If it says the version can be installed, check that it matches the version named in the report.

**Example prompt to copy**

```text
Use skill-security-check to review this before installation.
Material to check: [target].
If I provided a public GitHub link above, I authorize read-only access to that repository for this review.
Check whether it can read passwords or unrelated files, send data out, run scripts, enable extra tools automatically, or bypass approval. Do not install, run, or change the target. If you cannot read the whole folder safely, tell me what is missing.
Start with a one-sentence verdict on whether I can install it. Then give the version and scope you reviewed, evidence locations, anything you could not see, and my next step. Never echo a complete credential.
```

**Try it**: Set the target to “the short skill text below.” Add this line at the end of the prompt: “Organize the to-do list I provide.” A good report can comment on the text it read but will still say it lacks the files needed to clear the whole package for installation.

### 3. Check a GitHub code repository

**Review files**: [SKILL.md](github-repo-security-check/SKILL.md), [repo-threat-checklist.md](github-repo-security-check/references/repo-threat-checklist.md), and [search-patterns.md](github-repo-security-check/references/search-patterns.md).

1. Open the GitHub project and copy its home page link. For a project on your computer, give an AI with local file access the project folder path. Include the branch or commit if you know it. If you do not, write “Tell me which version you can review first.”
2. Copy the prompt below and fill in the target and what you want to check. For a private project, use an authorized local folder or connection. If access is limited, start with the files you are allowed to share.
3. Read the verdict and coverage gaps first. Address serious risks before running or integrating the project. If parts were missed, ask: “Which folders, dependencies, or settings do you still need?” If the project also loads a skill, plugin, or agent configuration, use the second security review on the **same version**.

**Example prompt to copy**

```text
Use github-repo-security-check for a read-only security review.
Repository: [GitHub repository link or local folder path].
Version and scope: [branch, commit, or subfolder; if unknown, write: tell me which version you can review first].
Focus: [for example, leaked keys, whether users can see someone else's data, or unexpected installation and build steps].
If I provided a public GitHub link, I authorize read-only access to that repository for this review. Do not install dependencies, build, run, or change the target.
Report the version and scope you reviewed, each risk with a file and line number, what to fix, and what you could not verify. If the project also loads AI skills or agent configurations, tell me which additional security review it needs.
```

**Try it**: If you do not have a project at hand, practice on [this repository](https://github.com/baibaigithub/agent-skill-check). Use `main` as the version and set the focus to “List the files first, then tell me what you can check and what does not apply.” For a budgeting app, you could ask “Can a signed-in user see someone else's bills? Do logs record account passwords?”

## Review limits

These skills perform static review: they read material and look for evidence. A report applies to the version and scope it names. Actual data transfers and permissions in production require separate verification.

If this helped you avoid a problem, consider giving the repo a ⭐ Star. You're also welcome to open an issue about a suspicious skill you've come across.
