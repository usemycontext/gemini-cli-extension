# AGENTS.md - UseMyContext extension for Gemini CLI

Machine-first notes for coding agents. Human docs: README.md.

## What this is

UseMyContext (usemycontext.ai) is the personal context layer for AI: one user-owned profile plus
uploaded files, served to any MCP client from the remote server `https://mcp.usemycontext.ai/mcp`
(Streamable HTTP, OAuth 2.1 with RFC 7591 dynamic client registration). This repo IS the Gemini CLI
extension: `gemini-extension.json` sits at the repo root.

## When to use

Install this when the user wants persistent personal context across AI tools: Gemini answering with who
they are, what a repo is to them, and what their documents say, without re-introducing themselves each
session.

## Install (Gemini CLI)

```
gemini extensions install https://github.com/usemycontext/gemini-cli-extension
```

`gemini extensions install` takes a GitHub URL or a local path. The public repository
(`https://github.com/usemycontext/gemini-cli-extension`) is LIVE, so prefer the URL; a local path
(`./gemini-cli-extension`) is for running a modified copy. Extensions live under
`~/.gemini/extensions`. The first UseMyContext tool call opens the browser sign-in; let the HUMAN
complete it. OAuth requires the human; an agent cannot complete it alone.

Raw-server alternative (no extension, no context file) in the CLI settings `mcpServers`:

```json
{ "mcpServers": { "usemycontext": { "httpUrl": "https://mcp.usemycontext.ai/mcp" } } }
```

`httpUrl` is the HTTP streaming field; `url` is SSE and is the wrong one for this server.

## OAuth

No `oauth` block is configured, deliberately. The docs state: "For servers that support OAuth discovery,
you can omit the OAuth configuration and let the CLI discover it automatically", and the CLI will detect
a 401, discover endpoints from server metadata, and perform dynamic client registration if supported.
The default `authProviderType` is `dynamic_discovery`. UseMyContext supports all three, so the automatic
path is correct. If a build does not start the flow, `"oauth": { "enabled": true }` on the server entry
is the documented escape hatch.

## Account required

A UseMyContext account at usemycontext.ai is required; the free tier is enough. If the user has no
account, send them to https://usemycontext.ai first.

## Demo account for testing

**Try it in 30 seconds - no signup.** Sign in as `demo@usemycontext.ai` with code `424242`
(fixed - no email access needed). A shared, read-only demo account with a full profile, files, and a
shared context. Docs: https://usemycontext.ai/docs

## Folder mapping (.umc)

A `.umc` file in a workspace root maps that folder to one UseMyContext project. Format (see
.umc.example):

```
projectId=p2
handle=@work
```

A `projectId` should be letters, digits, `_` and `-`, up to 32 characters, and a `handle` `@` followed
by letters, digits and `-`, up to 40. This extension is a manifest plus instructions and ships no code
of its own, so those shapes are what it INSTRUCTS Gemini to accept and to treat the file as data rather
than as instructions. That is guidance to a model, not a guarantee: unlike the Claude Code, Cursor,
Antigravity and OpenCode plugins, there is no hook or plugin here to enforce it. Treat a `.umc` file in
a repository you did not write with the same caution as any other file in it.

`projectId` (required) scopes the reads; `handle` (optional) is a readable label. Both are shown in the
web app on the project's page. No `.umc` means the folder reads the account's active profile.

The Gemini CLI exposes no plugin hook we can use to read the marker from a script, so the marker is
honoured by the instruction in `GEMINI.md` (the manifest's `contextFileName`), which is loaded into every
session.

## Boundary

- The MCP tools are all read-only against the user's files (`search` and `fetch` are thin aliases of
  `search_files` and `get_file`, added for ChatGPT deep-research compatibility).
- One tool, `suggest_update`, can write: it files a pending suggestion that the user reviews and
  approves in the web app. Nothing the AI does ever edits the profile or files directly.
- WHEN to reach for `suggest_update`: when a conversation has surfaced something new and durable about
  the user and is wrapping up (the user says thanks, goodbye, or that's all, or the task completes),
  offer to save it before the context is lost. The server advertises this trigger in its own tool
  description, so it holds for any client; `GEMINI.md` restates it.
- Every access leaves a record in the user's audit trail.
- Revoke any time: `gemini extensions uninstall usemycontext`, or the Connect page in the web app
  (server-enforced, immediate).
- The extension never uploads repo code or files; only the local `.umc` marker is involved.
