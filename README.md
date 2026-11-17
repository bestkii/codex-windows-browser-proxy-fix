# Codex on Windows: fixing slow in-app browser reads when using a local proxy

The built-in browser displayed pages normally, but Codex sometimes timed out when asked to read them. Even after a restart restored access, reading a simple page still took about **21 seconds**. On my Windows setup, passing my existing local proxy to Codex’s Node REPL through the Windows system config reduced repeated DOM reads to **12–42 ms** after a restart.

## When this might apply

- You use the **built-in browser** in Codex on Windows.
- Pages load visibly, but agent reads take about 20–30 seconds, time out, or a fresh chat reports `Browser is not available: iab`.
- You already run a trusted local HTTP proxy.

My test setup was Windows 11 24H2 (build 26100.6584), Codex desktop 26.1111.21308, Windows package 26.1111.2228.0, and Clash Verge. This is a workaround verified on one setup.

## What I found

Codex’s generated Node REPL configuration had no proxy environment variables. In a separate test with Codex’s bundled Node executable, an unauthenticated request to `ab.chatgpt.com` timed out without the explicit HTTP proxy. Through the existing proxy, it returned HTTP 403 in 773 ms. That response established reachability. It didn’t exercise Codex’s authenticated feature-policy flow.

The desktop log also recorded `IAB browser belongs to another task` during diagnosis. The fresh-chat error stopped reproducing in my acceptance test after the workaround. The log does not establish why the browser routing requests failed. [Related proxy timeout report](https://github.com/openai/codex/issues/33713) · [Related browser routing report](https://github.com/openai/codex/issues/61150)

## Workaround

1. Confirm the local **HTTP or mixed proxy port** in your proxy app. The example below uses `7897`. Substitute your own port.
2. With administrator access, add the settings below to `%ProgramData%\OpenAI\Codex\config.toml`. Back up any existing file and merge the keys into the existing table if it is already present. Keep the system config and its parent directories writable only by administrators or trusted system accounts. This file supplies defaults for all local Windows accounts using Codex. [Codex Windows config loader](https://github.com/openai/codex/blob/rust-v0.169.0/codex-rs/config/src/loader/mod.rs) · [Codex MCP environment settings](https://learn.chatgpt.com/docs/config-file/config-reference)

```toml
[mcp_servers.node_repl.env]
NODE_USE_ENV_PROXY = "1"
HTTP_PROXY = "http://127.0.0.1:7897"
HTTPS_PROXY = "http://127.0.0.1:7897"
NO_PROXY = "localhost,127.0.0.1,::1"
```

3. Fully quit and reopen the desktop app. Open a new local Codex chat and ask it to use the built-in browser to open `https://example.com/` and read the page text. Page visibility alone doesn’t verify agent access.

I tried the same settings in my user config first. Codex regenerated the Node REPL block on restart and removed them. The Windows system config survived the restart.

The settings apply to supported HTTP(S) requests from the Node REPL. The proxy still chooses the outgoing route, including any `DIRECT` rules. Keep the local proxy running while using the workaround. To roll back, restore the backup or remove only the four added entries, then restart Codex. [Node.js proxy behavior](https://nodejs.org/learn/http/enterprise-network-configuration)

## Results

| Test | Result |
| --- | --- |
| Before the workaround | One successful DOM read took **21.4 s** |
| After restart, same chat | **42 ms**, **14 ms**, **13 ms** |
| After switching chats and returning | **26 ms**, **12 ms**, **12 ms** |
| Brand-new chat, first connection | **3.95 s**, with no manual browser panel setup |
| Brand-new chat, three DOM reads | **381 ms**, **12 ms**, **11 ms** |

These were timings on one computer against `example.com`. Discovery, navigation, and DOM reads are separate operations. If you reproduce this on another version or setup, the most useful details are the app version, proxy mode, and whether a fresh chat can connect and read the page.
