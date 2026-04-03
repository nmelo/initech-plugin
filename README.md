# initech-plugin

Claude Code plugin for coordinating [initech](https://github.com/nmelo/initech) agent fleets.

## Install

```bash
claude plugin add nmelo/initech-plugin
```

## Skills

### fleet-coordinator

Teaches Claude Code how to observe, dispatch, and steer an initech multi-agent session. Covers status checking, work dispatch, QA routing, engineer selection, and anti-patterns.

Requires the initech MCP server (`initech_peek`, `initech_send`, `initech_status`, `initech_patrol` tools).

## License

MIT
