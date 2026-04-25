# AgentWatch Privacy Policy

_Last updated: 2026-04-25_

AgentWatch is a macOS menu bar app that detects AI agent processes running on your Mac. This is a short, plain-language explanation of what the app does and does not do with information about you.

## What AgentWatch reads from your Mac

To do its job, AgentWatch reads the following from the running system:

- The list of processes (via `sysctl(KERN_PROC_ALL)`)
- The command-line arguments of those processes (via `sysctl(KERN_PROCARGS2)`)
- Per-process memory and CPU usage (via `proc_pidinfo`)
- Per-process open network sockets — remote IP and port only (via `proc_pidinfo` + `proc_pidfdinfo`)
- Your Mac's total physical RAM (via `sysctl(HW_MEMSIZE)`) — used to color the menu bar dot

This information is read **on your Mac, by the app you're running**. It never leaves your device.

## What AgentWatch sends to the network

**One thing, and only one thing**: on every launch, AgentWatch fetches an updated copy of `signatures.json` from this repo:

```
https://raw.githubusercontent.com/medomar/agentwatch-public/main/signatures.json
```

That request is a plain HTTPS GET. It does not include any data about you, your Mac, or the processes you're running. The only thing GitHub sees is the request itself — same as any web browser hitting any public URL.

If the fetch fails (no network, captive WiFi, GitHub down), the app uses the version of `signatures.json` bundled inside the app at build time, or the most recent one cached locally. **AgentWatch works fully offline.**

### What the landing page sends

If you visit the landing page at <https://medomar.github.io/agentwatch-public/>, your browser makes **one** request to `api.github.com` to display the public download count of the DMG. No personal data is sent in that request — it's the same call you'd make running `gh api repos/medomar/agentwatch-public/releases` from your terminal. The result is cached in your browser's `localStorage` for 5 minutes to keep traffic low. The landing page does not include any other tracking, analytics, or third-party scripts.

## What AgentWatch stores on your Mac

- A cached copy of `signatures.json` at `~/Library/Caches/Mohaioo.AgentWatch/signatures.json` — used to keep the app working between launches and to pick up registry updates.
- Standard SwiftUI / app-state caches managed by macOS.

That's it. No history of what you ran. No logs of what the app detected. No usage data.

## What AgentWatch does NOT do

- ❌ No analytics, no tracking, no telemetry.
- ❌ No "anonymous" usage data, no crash reporting service, no advertising IDs.
- ❌ No data is sent to me (the developer) or to any third party.
- ❌ AgentWatch never sends your process list, command-lines, IP addresses, or any other system information off your device.
- ❌ AgentWatch does not access files outside its own cache directory.

## How to verify

Every claim above is checkable:

- The app's source code is the source of truth. The network code is in [`AgentWatch/Detection/RegistryFetcher.swift`](https://github.com/medomar/AgentWatch/blob/main/AgentWatch/Detection/RegistryFetcher.swift) — it makes one URLSession request per launch to the URL above and writes the response to the local cache. Nothing else.
- The detection rules are in [`signatures.json`](https://raw.githubusercontent.com/medomar/agentwatch-public/main/signatures.json) in this repo. Public, auditable.
- You can confirm the app is making no other network requests using **Little Snitch**, **LuLu**, **Charles**, or `tcpdump`.

If you spot any deviation from the above, please [open an issue](https://github.com/medomar/agentwatch-public/issues) and I'll fix it.

## Contact

The maintainer is [@medomar](https://github.com/medomar) on GitHub. Privacy questions, security disclosures, or "what does the app actually do?" follow-ups: [open an issue](https://github.com/medomar/agentwatch-public/issues) on this repo.
