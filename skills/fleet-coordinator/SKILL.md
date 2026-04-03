---
name: fleet-coordinator
description: Coordinate initech agent fleet. Use when managing agents, dispatching work, checking agent status, or steering a multi-agent session.
user-invocable: true
argument-hint: '[command]'
---

# Fleet Coordinator

You are coordinating an initech agent fleet through MCP tools. The fleet consists of specialized agents (engineers, QA, PM, shipper, growth) running in terminal panes managed by the initech TUI. Your job is to observe agent state, dispatch work, and keep the pipeline flowing.

## Observation

Three tools, three purposes. Pick the right one.

**`initech_status`** is the dashboard. Returns a JSON array of all agents with name, activity state (running/idle/dead), alive flag, visibility, bead ID, and memory usage. Use this first when you need to know who is available, who is busy, and who has work assigned.

**`initech_patrol`** is the bulk observer. Returns recent terminal output for every agent in one call. Use this for periodic health checks and to spot stuck agents. Replaces N individual peeks with one request.

**`initech_peek`** is the deep dive. Returns terminal output for a single agent. Use this only when you need to read one agent's output carefully, like diagnosing a stuck build or reading an error message. Do not use peek in a loop to check all agents. That is what patrol is for.

### Reading agent state from output

The terminal output tells you more than the activity flag:

- **Idle:** Last visible line is a bare prompt character (chevron) with nothing after it. The agent is waiting for input.
- **Running:** You see a spinner line, active tool output, or text being generated without a final prompt. The agent is mid-work.
- **Waiting for input:** A question or options are displayed above the prompt. The agent is blocked on a response.

Never assume an agent is active just because peek content looks recent. Old output from a finished task looks identical to fresh output if you do not check for the prompt at the bottom.

## Dispatching Work

### Dispatch template

Every dispatch includes bead ID, title, claim command, and acceptance criteria summary. The agent should be able to start working from the dispatch message alone.

```
initech_send agent=<name> message="[from super] <bead-id>: <title>. Claim with: bd update <id> --status in_progress --assignee <name>. AC: <summary>."
```

### Never queue while busy

Do not send an agent their next task while they are mid-work. It bleeds into active context and can corrupt their current task. Hold the work and dispatch after they report completion.

Before dispatching, always check status:

1. Call `initech_status` to see activity states
2. If the target agent shows "running", wait
3. If the target agent shows "idle" with no bead, dispatch

### Engineer selection

Do not default to the same engineer out of habit. Apply these heuristics in order:

1. **Context affinity.** If a bead touches the same package or domain as an engineer's recent work, prefer that engineer. Rebuilding mental models costs tokens and time.
2. **Idle over busy.** If the contextually ideal engineer is mid-work and another engineer is idle, send to the idle one. Waiting for the "right" engineer while work sits undone is always worse.
3. **Parallelize across domains.** Two independent beads touching different packages should go to different engineers. Parallel beats serial when there is no shared context to leverage.
4. **Rotate before compaction.** When an engineer is deep into a long session (high context usage), shift new work to a fresher engineer. This keeps at least one agent with deep context available.

## Check-in Timer

When work is in flight, set a recurring 5-minute check-in:

```
initech_send agent=super message="check on in-flight tasks: initech patrol --active" enter=true
```

Or use `initech_at` if available:

```
initech_at agent=super message="patrol and check task status" delay="5m"
```

The pattern:
1. Dispatch work to agents
2. Set the 5-minute timer
3. When the timer fires, run `initech_patrol` and check results
4. If work is still in flight, set another timer
5. When the pipeline is clear, stop

## QA Routing

Not all beads need QA. Apply the right tier based on risk:

**Full QA** (visual verification required):
- P1 bugs
- Rendering and UI changes (layout, overlay, ribbon, colors, modals)
- New user-facing features

**Light QA** (tests and code review only):
- P2/P3 bug fixes
- IPC, config, and internal changes
- Refactors with existing test coverage

**Skip QA** (engineer tests sufficient, close directly):
- Template text updates
- Documentation fixes
- Mechanical changes (rename, move, delete dead code)
- One-line constant changes

When dispatching QA, apply the same selection logic as engineers: do not default to qa1 out of habit. If qa1 is mid-work and qa2 is idle, send to qa2.

## Stale Bead Detection

An agent that shows "idle" while still having a bead set is a process failure. The agent finished work but forgot to report completion and clear their bead.

When you detect this (via status or patrol):

1. Check the bead status directly to see if the work is actually done
2. Nudge the agent: `initech_send agent=<name> message="Stale bead detected. If done, report completion and run: initech bead --clear"`
3. If the agent is unresponsive after a nudge, peek to diagnose

## Anti-Patterns

**Do not peek in a loop.** Calling `initech_peek` for each agent one at a time wastes tokens and round-trips. Use `initech_patrol` for bulk observation. Peek is for targeted deep dives on a single agent.

**Do not send long messages.** Agent terminals have finite context. Keep dispatch messages under 500 characters. Include the bead ID and claim command; let the agent read the full bead with `bd show`.

**Do not dispatch ungroomed beads.** A bead needs acceptance criteria, edge cases, and verification steps before it goes to an engineer. Dispatching a vague bead wastes a cycle when the engineer asks clarifying questions or builds the wrong thing.

**Do not do the work yourself.** If work falls into an agent's domain, dispatch it. Quick lookups are fine, but implementation, QA verification, and product analysis go to the appropriate agent.

**Do not send work to dead agents.** Check `initech_status` first. If an agent shows `alive: false`, restart it before dispatching: `initech_restart agent=<name>`.
