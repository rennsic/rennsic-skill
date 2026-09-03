# Rennsic VIN history skill

Rennsic cross-references a VIN against a registry of peer-to-peer rental
listings and returns the vehicle's commercial rental history: how many trips it
ran, how many separate owners listed it, and how that volume compares to the
rest of the fleet. Conventional vehicle history reports miss this entirely. A
car with hundreds of commercial rides can show a clean report.

Where renters left reviews, a lookup also carries what they reported about the
car: how many distinct issues, how much evidence that rests on, any
disclosures, and a few quoted findings. There is no score attached to it, by
decision, and no review data for a VIN means none is indexed rather than that
the car came back clean.

This repository packages that API as an Agent Skill, so an LLM agent can run VIN
lookups and read the results correctly.

## Install

Claude Code marketplace:

```
/plugin marketplace add rennsic/rennsic-skill
/plugin install rennsic-vin-history@rennsic
```

Manual, for any agent that reads skill directories:

```
cp -r skills/rennsic-vin-history ~/.claude/skills/
```

## Token

The API requires an active Pro or Dealer plan; Starter is dashboard only. Get
your token from the API console at https://rennsic.com/api/reference/ and
export it:

```
export RENNSIC_API_TOKEN="your-token"
```

Set `RENNSIC_BASE_URL` to point the skill at a different deployment. It defaults
to `https://rennsic.com`.

Treat the token as a credential: keep it out of source control, shared
transcripts, and client-side code.

## Bundled MCP server

The plugin bundles the Rennsic MCP server (`.mcp.json`, pointing at
https://rennsic.com/api/mcp/), which gives Claude Code the lookups as native
tools instead of shell commands (`vin_lookup`, `search_history`,
`credit_usage`, `request_report`), plus two prompts invoked by name:
`/mcp__rennsic__check_vin`, with VIN autocompletion from the account's own
paid lookups, and `/mcp__rennsic__account_status`. When the server is
connected, the skill defers to those tools and keeps only the reading rules.

Installing the plugin registers the server; Claude Code asks for approval on
first use, then signs in with OAuth in the browser. No token is shipped in the
plugin and none is needed for MCP. `RENNSIC_BASE_URL` points the bundled
server at a different deployment, the same variable the shell recipes use. To
use the skill without the server, decline the approval or add `rennsic` to
`disabledMcpServers` in Claude Code settings; the shell recipes keep working
on the API token alone.

Other MCP clients (Cursor, claude.ai, the Claude API) connect to the same URL
directly; the API console has setup snippets.

## Contents

| Path | What it is |
| --- | --- |
| `skills/rennsic-vin-history/SKILL.md` | Operating instructions the agent loads |
| `skills/rennsic-vin-history/reference.md` | Full response shapes and error codes |
| `.mcp.json` | The bundled MCP server registration |

## License

MIT. See [LICENSE](LICENSE).
