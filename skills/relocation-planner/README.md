# relocation-planner (Claude Agent Skill)

Compare retirement relocation / state-tax arbitrage by orchestrating the public planfi MCP — how much a move (e.g. CA→TX/FL) saves in state income tax, property tax, estate tax, and cost of living, and whether it's worth it.

It's a **thin orchestration layer** over the public **planfi MCP** (`https://ai.planfi.app/mcp`,
public, no auth) — all the math and financial logic live server-side. The skill itself bundles no
engine; it just gathers inputs and calls the tools.

### MCP bootstrap (one time)

If the planfi tools aren't connected yet, run:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

On **claude.ai**: Settings → Connectors → add a custom connector pointing at
`https://ai.planfi.app/mcp` (no auth). The skill also reminds you to do this if the tools are
missing when you invoke it.

## Install

### Quickest — skills.sh CLI (recommended)

```
npx skills add holdequity/planfi-relocation-planner
```

### Claude Code — copy the folder (zero prerequisites)

User-level (available in every project):

```
cp -r skills/relocation-planner ~/.claude/skills/
```

Or project-level (only this repo/project):

```
mkdir -p .claude/skills && cp -r skills/relocation-planner .claude/skills/
```

Restart Claude Code (or start a new session). The skill auto-loads by its description.

### Claude Code — as a plugin (shareable one-liner)

```
/plugin marketplace add holdequity/planfi-relocation-planner
/plugin install relocation-planner@planfi
```

### claude.ai — upload as a skill

1. Zip the folder: `cd skills && zip -r relocation-planner.zip relocation-planner`
2. In claude.ai go to Settings → Capabilities → Skills and upload the zip.

## Notes & honest caveats

- All decimals are fractions (24% → `0.24`); all figures are today's (real, inflation-adjusted)
  dollars; tax brackets/limits are approximate ~2026 values.
- The skill surfaces every assumption the server reports (`disclosures.key_assumptions`) so you can
  correct any silent default.
- Not financial advice. Planning estimates only.

See `SKILL.md` for the full instructions, exact tool params, and output format.
Source + issues: <https://github.com/holdequity/planfi-relocation-planner>.

## License

MIT — see [LICENSE](./LICENSE).
