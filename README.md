# Cycles IQ API — Agent Skill

A free skill that teaches AI agents how to use the [Cycles IQ API](https://api.cyclesiq.com/specs/index.html?url=/apidocs/v1/swagger.json):
analysing your own time series: finding dominant cycles, scanning the cycle spectrum, applying DSP
filters and CRSI, getting cycle consensus scores, storing datasets, and reading every result correctly.

The skill is plain Markdown (`skills/cycle-api/SKILL.md` plus references), so any agent that
loads skill files can use it. It installs directly as a Claude Code plugin.

## Install (Claude Code)

```
/plugin marketplace add whentotrade/cycle-api-skill
/plugin install cycle-api@cycle-api-skill
```

Or for a team, in `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "cycle-api-skill": {
      "source": { "source": "github", "repo": "whentotrade/cycle-api-skill" }
    }
  },
  "enabledPlugins": {
    "cycle-api@cycle-api-skill": true
  }
}
```

## You need an account and a key

Every call needs a Cycles IQ account. Sign up free at app.cyclesiq.com: a new account starts with a
7-day trial and then continues on the Free plan, which covers every analysis route with your own data
and 3 stored datasets. Confirm your e-mail address, create a key on the API page of the app (FSC
members: app.cycles.org) and send it in the `X-API-Key` header.

## MCP

The same analyses are an MCP server at `https://api.cyclesiq.com/mcp`, for your own data. Add it as a
connector in Claude or ChatGPT and sign in with your Cycles IQ account (you can sign up right there),
or connect with a key:

```
claude mcp add --transport http cycles-iq https://api.cyclesiq.com/mcp --header "X-API-Key: <key>"
```

## What's inside

```
skills/cycle-api/
├── SKILL.md                     supplying data, key levels, endpoint map, workflows, results, pitfalls
└── references/
    ├── endpoints.md             every public endpoint: parameters, body, answer, when to use it
    ├── request-examples.md      curl, Python, JavaScript and C#, with rate-limit handling
    ├── phase-guide.md           phase strings, phase scores, average vs current groups
    ├── consensus-guide.md       the Cycle Consensus score and how to read it
    └── crsi-signals.md          how the API derives crsiScore / crsiSignal
```

## Bring your own data

The API analyses your own series. Send the values with each call, or store them once and name them:

```
PUT  /api/datasets/MYSERIES                          [{"dateUnix": 1758672000, "close": 101.2}, ...]
POST /api/cycles/CycleScanner?datasetid=MYSERIES     (no body)
```

## Building cycle applications

This skill covers the API. Charting, dashboards, composite cycle reconstruction and
continuous scoring methods live in the companion **cycle-tools** skill set.

## License

MIT. See `LICENSE`.
