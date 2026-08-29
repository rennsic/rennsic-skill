# Rennsic VIN history skill

Rennsic cross-references a VIN against a registry of peer-to-peer rental
listings and returns the vehicle's commercial rental history: how many trips it
ran, how many separate owners listed it, and how that volume compares to the
rest of the fleet. Conventional vehicle history reports miss this entirely. A
car with hundreds of commercial rides can show a clean report.

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

The API requires a Pro or Dealer subscription. Get your token from the API
console at https://rennsic.com/api/reference/ and export it:

```
export RENNSIC_API_TOKEN="your-token"
```

Set `RENNSIC_BASE_URL` to point the skill at a different deployment. It defaults
to `https://rennsic.com`.

Treat the token as a credential: keep it out of source control, shared
transcripts, and client-side code.

## MCP alternative

The same API is exposed as a remote MCP server at
https://rennsic.com/api/mcp/, which gives an MCP client the lookups as
native tools instead of shell commands. The API console has setup snippets for
Claude Code, Cursor, and the Claude API.

## Contents

| Path | What it is |
| --- | --- |
| `skills/rennsic-vin-history/SKILL.md` | Operating instructions the agent loads |
| `skills/rennsic-vin-history/reference.md` | Full response shapes and error codes |

## License

MIT. See [LICENSE](LICENSE).
