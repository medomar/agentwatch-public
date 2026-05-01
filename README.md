# AgentWatch

**See which AI agents are running on your Mac.** A tiny menu bar app that shows you what's using your CPU and RAM — Claude, Cursor, Ollama, Aider, and 70+ others — and lets you pause or kill them in two clicks.

- 🌐 **Website:** [medomar.github.io/agentwatch-public](https://medomar.github.io/agentwatch-public/)
- ⬇️ **Download (macOS):** [AgentWatch.dmg](https://github.com/medomar/agentwatch-public/releases/download/app-v1.0.4/AgentWatch.dmg) — latest stable, `app-v1.0.4`
- 🔒 **Privacy:** [PRIVACY.md](./PRIVACY.md) — one JSON fetch on launch, nothing else leaves your Mac
- 🐛 **Issues / feedback:** [open an issue](https://github.com/medomar/agentwatch-public/issues)

---

## What this repo is

This is the **public detection registry** for AgentWatch — the data file (`signatures.json`) the app downloads on launch to know which processes count as AI agents.

- Update this file → AgentWatch users get the new detection on their next launch.
- No app update, no notarization roundtrip.

```
agentwatch-public/
├── signatures.json           — the registry the app fetches
├── signatures.schema.json    — JSON Schema for the registry format
├── docs/                     — landing page (GitHub Pages)
└── PRIVACY.md
```

## How the app uses it

On launch, AgentWatch fetches:

```
https://raw.githubusercontent.com/medomar/agentwatch-public/main/signatures.json
```

1. Reads cached copy from `~/Library/Caches/<bundle-id>/signatures.json`, else falls back to the version bundled in the app.
2. Fetches the live URL in the background (10s timeout). Newer `registryVersion` → saved to cache.
3. **Applied on next launch**, not mid-session — keeps detection consistent within a session.
4. Fetch fails (offline, GitHub down) → nothing breaks. The app keeps using whatever it loaded at launch.

## How detection works

AgentWatch identifies a process as an AI agent via three clues, first match wins:

1. **Process name** — exact match against `kp_proc.p_comm` (capped at 16 chars by macOS). Used for CLIs with their own binary (`claude`, `aider`, `ollama`).
2. **Argv keywords** — substring match in the joined command-line. Only checked for processes whose name is in `argumentRuntimePrefixes` (`node`, `python`, VS Code's `Code Helper`). This is how editor extensions and SDK scripts are detected.
3. **Network** — open socket to a known localhost port (Ollama, LM Studio, vLLM, Jan…) or to a known cloud-provider IP range.

Schema extras: `pathAnchors` (stricter argv match), `extensionPaths` (editor extension install paths), `editorExtensionFolders` (parent-app resolution). Full schema: [`signatures.schema.json`](./signatures.schema.json).

## Contributing

PRs welcome to add a new tool or fix an existing entry. The registry is **curated** — every PR is reviewed. CI validates against the schema, so a malformed file won't reach review.

Minimum entry:

```json
{
  "id": "your-tool-id",
  "displayName": "Your Tool",
  "agentType": "unknown",
  "icon": { "glyph": "◌", "iconURL": null },
  "processNames": ["yourtool"],
  "confidence": "high"
}
```

Required: `id`, `displayName`, `agentType`, `confidence`, and at least one matching mechanism (`processNames`, `argvKeywords`, `pathAnchors`, `extensionPaths`, or `ipRanges`).

### Conventions

- **`id`** — kebab-case, never reused once shipped.
- **`processNames`** — cap at 16 chars (macOS `p_comm` limit). Longer names need `argvKeywords` or `pathAnchors`.
- **`confidence: high`** — verified against a real running process or officially documented. `medium` = read the docs, didn't run it. `low` = unverified guess.
- **No generic keywords.** `chat`, `gpt`, `llm`, `ai` are banned — they collide with non-AI tools. Borderline keywords need `pathAnchors`.

### Verifying locally

```sh
ps -axo pid,comm,command | grep your-tool      # find a running process
ps -axo comm= -p <pid> | head -c 16            # check the 16-char comm
ps -o command= -p <pid>                        # see full argv
```

### Won't accept

- Browser-based AI (chat.openai.com, claude.ai web) — no inspectable process.
- Mobile AI apps — AgentWatch is macOS-only.
- Generic English words as keywords without anchors.
- Entries you haven't actually verified detect anything.

## License

[MIT](./LICENSE) — use the registry data and schema however you want, including in your own AI-process detection tooling.

---

Maintained by [@medomar](https://github.com/medomar).
