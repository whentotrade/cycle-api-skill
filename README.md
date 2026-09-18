# Cycle Analysis API — Agent Skill

A free skill that teaches AI agents how to use the [Cycle Analysis API](https://api.marketzeitgeist.com/specs/index.html?url=/apidocs/v1/swagger.json):
loading market data, finding dominant cycles, scanning the cycle spectrum, applying DSP
filters and CRSI, getting cycle consensus scores, and reading every result correctly.

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

## You need an API key

Every request carries `?api_key=<your key>`. <!-- TODO(Lars): add the sign-up / key request link -->

## What's inside

```
skills/cycle-api/
├── SKILL.md                     endpoint map, standard pipeline, response schemas, pitfalls
├── pipeline-tester.html         try the full pipeline in a browser
└── references/
    ├── endpoints.md             every endpoint: parameters, bodies, responses
    ├── request-examples.md      working C# / JavaScript / Python code
    ├── phase-guide.md           phase strings, phase scores, average vs current groups
    ├── consensus-guide.md       the Cycle Consensus score and how to read it
    └── crsi-signals.md          how the API derives crsiScore / crsiSignal
```

Try the pipeline tester locally:

```bash
cd skills/cycle-api
python3 -m http.server 7842
# open http://localhost:7842/pipeline-tester.html
```

## The pipeline in one picture

```
SearchSymbols → EnsureCompleteDataset → (WaitUntilUpdateCompleted) → GetDatasetSeries → analysis endpoint
```

There is no single "update dataset" call. `EnsureCompleteDataset` and
`WaitUntilUpdateCompleted` work as a pair.

## Building cycle applications

This skill covers the API. Charting, dashboards, composite cycle reconstruction and
continuous scoring methods live in the companion **cycle-tools** skill set.

## License

MIT. See `LICENSE`.
