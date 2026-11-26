# UseMyContext for Gemini CLI

[MIT license](./LICENSE) | [usemycontext.ai](https://usemycontext.ai) | [Docs](https://usemycontext.ai/docs) | [`usemycontext` SDK on npm](https://www.npmjs.com/package/usemycontext)

Own your context, for every AI app. UseMyContext keeps who you are, what you are working on, your files and your
calendar in one place, and any AI client reads it over MCP. This extension brings that into the Gemini CLI: sign in
once and Gemini answers with your context already in the room - who you are, what this project is to you,
what your documents say. And it does not wait to be asked: when a question touches you personally, Gemini
is guided to check your profile and documents before saying it does not know about you, and when a
conversation wraps up having learned something durable about you, it is guided to offer to save that
before the context is lost.

## What it looks like

```
cd ~/code/acme-api
gemini

> Who am I, and what is this project to me?

  This folder is mapped to your @work context. You're a backend engineer
  at Acme, this repo is the payments API you own, and your notes say the
  current focus is the v2 billing migration. Want me to pull the migration
  checklist from your files?
```

No pasting your background into every session. The context is already there, scoped to the folder you
are in.

## Install

`gemini extensions install` takes a GitHub URL or a local path. The public repository is live, so the
URL is the one-step route:

```
gemini extensions install https://github.com/usemycontext/gemini-cli-extension
```

A local path works too, if you want to run a modified copy: clone or download this directory and point
the CLI at it.

```
gemini extensions install ./gemini-cli-extension
```

Installed extensions live under `~/.gemini/extensions`. Start `gemini`, and the first UseMyContext tool
call opens the browser sign-in.

You will need a UseMyContext account. The free tier is enough to try this. No account yet? Create a free
one at [usemycontext.ai](https://usemycontext.ai) first.

## Try it in 30 seconds

**No signup needed.** At the browser sign-in, use `demo@usemycontext.ai` with code `424242`
(fixed - no email access needed). A shared, read-only demo account with a full profile, files, and a
shared context. Then ask Gemini "what do you know about me?". Docs:
[usemycontext.ai/docs](https://usemycontext.ai/docs)

## What is in this extension

| File | What it does |
|---|---|
| `gemini-extension.json` | The manifest: `name`, `version`, `mcpServers`, `contextFileName` |
| `GEMINI.md` | The context file loaded into every session (mapping rule and both session triggers) |
| `.umc.example` | A ready-made folder mapping template |

## How sign-in works

UseMyContext does not hand out a static client id and secret. It runs an OAuth 2.1 authorization server
with **dynamic client registration** (RFC 7591): the client registers itself at first contact.

**How the Gemini CLI handles it**
([MCP server configuration](https://geminicli.com/docs/tools/mcp-server/),
[extensions](https://google-gemini.github.io/gemini-cli/docs/extensions/)):

- `httpUrl` is the field for an HTTP streaming endpoint; `url` is the SSE field. The UseMyContext server
  speaks Streamable HTTP, so the manifest uses **`httpUrl`**.
- The docs state: "For servers that support OAuth discovery, you can omit the OAuth configuration and let
  the CLI discover it automatically." The CLI will automatically "Detect when a server requires OAuth
  authentication (401 responses), Discover OAuth endpoints from server metadata, Perform dynamic client
  registration if supported". The default `authProviderType` is `dynamic_discovery`.
- UseMyContext answers an unauthenticated MCP call with exactly the 401 that triggers this, publishes its
  authorization-server metadata, and supports RFC 7591 registration. So this extension ships **no**
  `oauth` block, which is the documented path for our case.

**If a build of the CLI does not start the sign-in on its own**, `oauth.enabled` is a documented field
and forcing it is the supported escape hatch. Add this to the server entry in the manifest:

```json
"oauth": { "enabled": true }
```

**Manifest fields.** `gemini-extension.json` is documented with `name`, `version`, `mcpServers`,
`contextFileName` and `excludeTools`. A `description` field is not documented, so the manifest does not
carry one.

## One folder, one context

A work repo reads your @work profile. A side project reads your @personal one. Same terminal, different
context, chosen by the folder you are in.

To bind a folder to a project, drop a `.umc` file in the workspace root
([`.umc.example`](./.umc.example) is a ready-made template):

```
projectId=p2
handle=@work
```

A `projectId` should be letters, digits, `_` and `-`, up to 32 characters, and a `handle` `@` followed
by letters, digits and `-`, up to 40. This extension is a manifest plus instructions and ships no code
of its own, so those shapes are what it INSTRUCTS Gemini to accept and to treat the file as data rather
than as instructions. That is guidance to a model, not a guarantee: unlike the Claude Code, Cursor and
OpenCode plugins, there is no hook or plugin here to enforce it. Treat a `.umc` file in
a repository you did not write with the same caution as any other file in it.

`projectId` is what scopes the reads. `handle` is an optional readable label. Find both in the web app on
the project's page. With no `.umc`, the folder reads your account's active profile.

`GEMINI.md` is this extension's context file, so the mapping rule is loaded into every session and Gemini
passes `projectId` on your behalf. The Gemini CLI has no plugin hook we can use to read the marker
ourselves, so the marker is honoured by that instruction rather than by a script.

## What you get

The UseMyContext MCP server at `https://mcp.usemycontext.ai/mcp` (Streamable HTTP, OAuth 2.1), with the
tools `profile`, `list_profiles`, `info`, `account`, `list_files`, `search_files`, `get_file`, `search`,
`fetch` (thin aliases of `search_files`/`get_file` for ChatGPT deep-research compatibility), `ask_docs`,
`query_table`, `suggest_update`, `shared_context` - all read-only against your files. One tool,
`suggest_update`, can write. Your answers to the first setup questions save straight to your profile;
everything else waits for your approval in the web app. Nothing the AI does ever touches your files, and
every access leaves a record in your audit trail. When a conversation is wrapping up having surfaced something new and durable about you,
Gemini is guided to offer that save before the context is lost - so the loop closes at both ends: your
context is read at the start of a session, and what the session taught is offered back at the end.

## Disconnecting and revoking

- Uninstall the extension with `gemini extensions uninstall usemycontext`.
- Or revoke from the UseMyContext web app: the Connect page lists every AI connected right now, each with
  its own Disconnect button, plus a "Disconnect all".

Revoking is enforced on the server: the token dies immediately, everywhere. Nothing needs cleaning up
locally.

## What it can and cannot do

- Reads: your profile and the files you uploaded to UseMyContext, scoped to the connected or mapped
  project.
- Writes: nothing to your files. The one write tool, `suggest_update`, saves your answers to the first
  setup questions straight to your profile; everything else waits for your approval in the web app.
- Every access leaves a record in your audit trail.
- You can revoke access at any time, by uninstalling the extension or from the web app's Connect page.
- The extension never uploads your repo code or files. The only local thing involved is the `.umc` marker.
- It does not change your Google account, your Gemini CLI login, your billing, or which model runs.

## Links

- Docs: [usemycontext.ai/docs](https://usemycontext.ai/docs)
- Web app: [usemycontext.ai](https://usemycontext.ai)
- MCP endpoint: `https://mcp.usemycontext.ai/mcp`
