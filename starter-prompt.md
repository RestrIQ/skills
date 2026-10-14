Install the RestrIQ skill for this agent.

Source: https://github.com/restriq/skills — copy the folder skills/restriq/ (SKILL.md and references/) unchanged.
Destination: your skills directory. Claude Code: ~/.claude/skills/restriq/ · Codex: ~/.codex/skills/restriq/ · Antigravity: ~/.gemini/config/skills/restriq/ · any other agent: wherever it loads SKILL.md folders.
If I ask for project-local: Claude Code: .claude/skills/restriq/ · Antigravity: .agents/skills/restriq/ · Codex: global only.
Fetch it with `git clone --depth 1 https://github.com/restriq/skills` into a temporary directory, or download the raw files; do not edit them. Replace any existing restriq/ folder at the destination, then delete the temporary clone.
When done, tell me the destination path and the first sentence of the skill's description, then say "RestrIQ skill installed".
