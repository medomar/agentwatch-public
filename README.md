# agentwatch-public

Public registry of AI agent detection signatures for [AgentWatch](https://github.com/medomar/AgentWatch) — the macOS menu bar app that shows which AI agents are running on your machine, how much RAM and CPU they're using, and lets you pause or kill them in two clicks.

This repository holds **`signatures.json`** — the data file the AgentWatch app fetches on every launch to know which processes count as AI agents and how to identify them. Update this file and AgentWatch users get the new detection rules without an app update.

## What's in here

```
agentwatch-public/
├── signatures.json           — the registry (the file the app fetches)
├── signatures.schema.json    — JSON Schema describing the registry format
└── README.md
```

## How AgentWatch uses this

On launch, AgentWatch fetches the raw URL of `signatures.json` from this repo:

```
https://raw.githubusercontent.com/medomar/agentwatch-public/main/signatures.json
```

Behaviour:

1. The app reads the cached copy from `~/Library/Caches/<bundle-id>/signatures.json` if one exists, else falls back to the version bundled inside `AgentWatch.app/Contents/Resources/signatures.json`.
2. In the background, the app fetches the live URL with a 10-second timeout. If the fetched `registryVersion` is newer, it's saved to the cache.
3. The fetched registry is **applied on the next launch**, not mid-session — keeps detection consistent within a session.
4. If the fetch fails (offline, GitHub down), nothing breaks. The app uses whatever it loaded at launch.

This means: a PR merged into this repo today is in users' apps tomorrow, with no notarization roundtrip and no force-update.

## How detection works

AgentWatch identifies a process as an AI agent via three "clues" applied in order — first match wins:

1. **Process name** — exact match against `kinfo_proc.kp_proc.p_comm` (the truncated process name macOS exposes — capped at 16 chars). Used for CLIs that have their own binary (`claude`, `aider`, `ollama`).
2. **Argv keywords** — substring match in the joined command-line. Only checked for processes whose name is in `argumentRuntimePrefixes` (`node`, `python`, VS Code's `Code Helper`). This is how VS Code / Cursor extensions and SDK-using scripts are detected.
3. **Network** — open socket to a known localhost port (Ollama, LM Studio, vLLM, Jan…) or to a known cloud provider IP range.

A few extras the schema supports:

- **`pathAnchors`** — stricter than `argvKeywords`. A process matches only if at least one anchor is present in argv. Used for collision-prone names (e.g. `goose` the SQL migrator vs Block's Goose CLI — both binaries are named `goose`, only Block's has `block-goose` in its install path).
- **`extensionPaths`** — substring matches characteristic of an editor extension's install location (`.vscode/extensions/anthropic.claude-code-`).
- **`editorExtensionFolders`** — top-level mapping that lets the parent-app resolver know `/.vscode/extensions/` belongs to "Visual Studio Code" even when argv[0] is a generic `Code Helper (Plugin)` binary. Mirrors for `.cursor/extensions/` and `.kiro/extensions/`.

Full schema in [`signatures.schema.json`](./signatures.schema.json).

## Contributing

Pull requests are welcome to add a new AI tool or correct an existing entry. **The registry is curated** — every PR is reviewed before merge. CI validates the JSON against the schema automatically, so a malformed file won't even reach review.

### Adding a new AI agent

Open a PR adding one provider entry to `signatures.json`. At minimum:

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

Required fields: `id`, `displayName`, `agentType`, `confidence`, and **at least one** matching mechanism (`processNames`, `argvKeywords`, `pathAnchors`, `extensionPaths`, or `ipRanges`).

Pick the `agentType` enum value from the schema. If your tool doesn't fit a named category, use `"unknown"` — the UI will still display it correctly with a generic icon.

### Conventions worth knowing

- **`id`** is kebab-case, lowercase, never reused. Once an ID ships in a release, treat it as permanent.
- **`processNames`** entries cap at 16 characters because that's what macOS exposes via `kp_proc.p_comm`. Longer names will never match — for those, use `argvKeywords` or `pathAnchors` against the full executable path.
- **`confidence: high`** means you've verified the signature against a real running process, or it's officially documented. Use `medium` for "I read the docs but didn't run it" and `low` for "I think this is right but it's unverified."
- **Avoid generic keywords.** `chat`, `gpt`, `llm`, `ai` are all banned — they collide with non-AI tools. If a keyword is borderline, use `pathAnchors` to disambiguate.
- **Aliases** (`aliases: ["OldName"]`) are for renames. If the tool was previously known under another name, list it here so the UI can fall back gracefully for users on older installs.

### How to verify a signature locally

If you have AgentWatch built locally and want to verify your new entry detects something:

```sh
# Find a running process you want to detect
ps -axo pid,comm,command | grep your-tool-name

# Confirm the comm fits in 16 chars
ps -axo comm= -p <pid> | head -c 16

# Check what argv looks like
ps -o command= -p <pid>
```

Then run AgentWatch from Xcode and confirm your tool appears in the list.

### What we won't accept

- **Browser-based AI tools** (chat.openai.com, claude.ai web). Those don't run as inspectable processes.
- **Mobile AI apps**. AgentWatch is macOS-only.
- **Generic English words as keywords** without anchors. `together`, `instructor`, `guidance` all need `pathAnchors`.
- **Anything you haven't actually verified detects something** — `confidence: low` is allowed but you should still describe in the PR what you observed.

## License

The registry data (`signatures.json`) and schema are licensed under [MIT](./LICENSE) — use it however you want, including in your own AI-process detection tooling. The data is not a trademark of any of the AI products it identifies; it's a fingerprint catalogue for detection purposes.

## Related

- **[AgentWatch](https://github.com/medomar/AgentWatch)** — the macOS app that consumes this registry. (private during development; will go public alongside v1.0 distribution.)

---

Maintained by [@medomar](https://github.com/medomar). Questions, PRs, complaints → [open an issue](https://github.com/medomar/agentwatch-public/issues).
