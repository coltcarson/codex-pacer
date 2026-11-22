# Security review: is Codex Pacer safe to run?

Reviewed 2026-11-22 against version 4.6.2 (build 92) of this fork.

**Verdict: nothing malicious was found, and the code does what the README says. It is an invasive tool by design, though, so "safe" depends on whether you accept what it touches.**

## Scope and limits

This was a read-only source review of the Mac app and the shared Swift engine.

- The app was not built or run.
- The released binaries (DMG, Windows installers) were not inspected.
- Git history was not inspected.
- The Windows shell and most of the roughly 100 engine files were only sampled by search, not read in full.
- Of the scripts, only the `make build` path (`scripts/native/build-island.sh`, `scripts/native/swiftpm-environment.sh`) was read. The test and release scripts were not.

## What was checked and found clean

- **No spyware capabilities.** There is no accessibility access, screen capture, keystroke or clipboard reading, analytics, or launch agent. The only persistence is the standard optional login item (`SMAppService.mainApp`).
- **Outbound hosts are all expected.** The app contacts `claude.ai`, `api.anthropic.com`, `chatgpt.com` (including `codex-cloud-backend.chatgpt.com`), `api.deepseek.com`, `platform.deepseek.com`, `www.workbuddy.cn`, `www.workbuddy.ai`, and GitHub for updates. The other links in the source (`pacer.bhgmz.com` download mirror, `buymeacoffee.com`, `qm.qq.com`) only open in the browser.
- **No foreign keychain reads.** The app only reads and writes its own keychain items (`Codex Pacer Claude Quota`, `Codex Pacer DeepSeek balance`). The code path for reading Claude Desktop's keychain is hard-wired off in production (`MacClaudeCredentials.swift`: the password provider always throws).
- **Hook scripts are local-only.** `claude_hook.py` and `codex_hook.py` use only the Python standard library, make no network calls, and write identifiers and timestamps to a local spool file. Prompts and tool input are dropped.
- **The build path is benign.** `make build` copies sources to `/private/tmp`, builds, and ad hoc signs. The one dependency, Sparkle 2.10.0, is pinned by revision in `native/Package.resolved`. GitHub Actions are pinned by commit hash.

## What it does to your machine

1. **Edits your agents' configuration.** It adds hooks, a status-line wrapper and telemetry environment keys to Claude Code's `settings.json`, and installs a hook plugin into Codex. With the progress ring on, it also adds a fixed line of context to typed Claude prompts. The status-line wrapper re-runs your previous status-line command through a shell.
2. **Uses your existing logins.** It reads `~/.claude/.credentials.json` or `CLAUDE_CODE_OAUTH_TOKEN`, and for the opt-in Dot feature asks the Codex CLI for the ChatGPT token. In the code read, these tokens are sent only to the vendor's own hosts.
3. **Runs code on your SSH hosts automatically.** For hosts set up in Codex or Claude Desktop, it runs Python over SSH and edits the Claude settings there. It uses `BatchMode=yes`, `StrictHostKeyChecking=yes` and `ForwardAgent=no`.
4. **Opens a local port.** It listens on `127.0.0.1:4319` for Claude telemetry, loopback only.
5. **Calls private web endpoints.** Sign-in happens in the app's own web view, and it then reads undocumented `claude.ai`, DeepSeek and WorkBuddy endpoints. This is not a security hole, but it may break or conflict with those services' terms.

## Trust caveats

- **Updates come from upstream, not this fork.** The Sparkle feed (`SUFeedURL`) and signing key (`SUPublicEDKey`) in `native/Info.plist` belong to `RyanZhangNTU/codex-pacer`, so even a self-built copy will offer that maintainer's future binaries. Updates are signature-checked and are not installed automatically (`SUAutomaticallyUpdate` is false).
- **The Windows installer is not code signed.** Only the updater's own Ed25519 signature check protects it.
- **Released binaries are unverified.** Nothing in this review proves that the downloadable DMG or installers match this source.

## Lowest-risk way to run it

Build from source with `make build` on a Mac, leave SSH hosts and Dot off, and decline updates until they have been reviewed.
