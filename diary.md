# NanoClaw Diary

## 2026-04-19 — Initial Setup & WhatsApp Bot Working

### What was done

Synced the forked repo (`br3no/nanoclaw`) with upstream (`qwibitai/nanoclaw`), merged all skill branches, and completed full first-time setup including container runtime, credentials, and WhatsApp channel.

### WhatsApp configuration

- Bot name: **Kai**, trigger: `@Kai`
- Dedicated number: separate SIM, not personal number
- Mode: DM with bot, no trigger required (main channel)
- JID format: `@lid` (newer WhatsApp protocol — `@s.whatsapp.net` didn't match)

### Credential system: OneCLI

OneCLI runs locally at `http://172.17.0.1:10254` (dashboard) / `:10255` (HTTPS proxy gateway). The proper integration uses `@onecli-sh/sdk`'s `applyContainerConfig()`, which:
- Sets `HTTPS_PROXY` pointing to the OneCLI gateway
- Mounts a CA certificate for HTTPS interception
- Sets `CLAUDE_CODE_OAUTH_TOKEN=placeholder`

The Anthropic credential (Claude Code OAuth token from `claude setup-token`) is stored in OneCLI as an `anthropic`-type secret. The proxy intercepts HTTPS traffic to `api.anthropic.com` and injects the real token.

**Gotcha — token format validation:** Claude Code validates that `CLAUDE_CODE_OAUTH_TOKEN` starts with `sk-ant-` before making any API call. The literal string `placeholder` fails this check and causes exit code 1 before the proxy even gets a chance to replace it. Fixed by using a properly-formatted placeholder: `sk-ant-oat01-placeholder-aaa...`.

### Root cause of "Claude Code process exited with code 1"

The container entrypoint starts as root (needed to bind-mount `/dev/null` over `.env` to prevent secrets leaking into the container). After the mount, it drops privileges via `setpriv` to `RUN_UID`.

`container-runner.ts` skipped setting `RUN_UID` when the host UID was 1000, assuming that matched the container's `node` user and no drop was needed. But the entrypoint *always* starts as root regardless, so containers were running as root throughout.

Claude Code refuses `--dangerously-skip-permissions` as root for security reasons → exit code 1 immediately.

**Fix:** always set `RUN_UID` for main containers when the host is non-root, even when `hostUid === 1000`.

### Other fixes along the way

- `src/container-runner.ts`: replaced manual gateway URL derivation with proper `OneCLI.applyContainerConfig()`
- `src/config.ts`: added `ONECLI_API_KEY` export
- `src/channels/whatsapp.ts`: `ownsJid()` extended to accept `@lid` JIDs
- `container/Dockerfile` entrypoint: made `.env` bind-mount non-fatal (`|| true`) for Docker compatibility
- `setup/verify.ts`: added `ONECLI_URL` to credentials check
- `.env`: `ASSISTANT_HAS_OWN_NUMBER=true` to suppress the "Kai: " prefix (only needed for shared numbers)
- SQLite: updated registered group JID from `@s.whatsapp.net` to `@lid` format
