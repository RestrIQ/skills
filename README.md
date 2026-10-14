<!-- Copyright © RestrIQ. All rights reserved. -->

# RestrIQ skill

This is an agent skill that teaches a coding agent how to integrate RestrIQ into an eCommerce, POS, or ERP system. RestrIQ blocks illegal shipments before checkout.

## Install

Paste this into your coding agent:

```text
Install the RestrIQ skill for this agent.

Source: https://github.com/restriq/skills — copy the folder skills/restriq/ (SKILL.md and references/) unchanged.
Destination: your skills directory. Claude Code: ~/.claude/skills/restriq/ · Codex: ~/.codex/skills/restriq/ · Antigravity: ~/.gemini/config/skills/restriq/ · any other agent: wherever it loads SKILL.md folders.
If I ask for project-local: Claude Code: .claude/skills/restriq/ · Antigravity: .agents/skills/restriq/ · Codex: global only.
Fetch it with `git clone --depth 1 https://github.com/restriq/skills` into a temporary directory, or download the raw files; do not edit them. Replace any existing restriq/ folder at the destination, then delete the temporary clone.
When done, tell me the destination path and the first sentence of the skill's description, then say "RestrIQ skill installed".
```

## Claude Code marketplace (alternative)

```
/plugin marketplace add restriq/skills
/plugin install restriq@restriq
```

## What the skill covers

- Credentials and hosts: the Key ID and Secret, the fixed console token endpoint, and a configurable Check API base URL for RestrIQ-hosted or self-hosted merchants, with the token's `aud` tying it to one host
- Token lifecycle: exchanging the key for a token, caching and refreshing it, and pinning the key to the storefront's `site`
- The check request and reading the response: destination, UPCs, `shippable`, `reasons`, and `modifications`
- Unknown UPCs and catalog items: the `not_found` decision, `unknown_upc_shippable`, and submitting product records
- Errors and where the check belongs: every documented error code, and placing the check in checkout, POS, and ERP or order systems

## Links

- Website: https://www.restriq.com
- API Keys page in the console: https://console.restriq.net/api-keys
