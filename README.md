# readable-tool-calls

A [Claude Code](https://claude.com/claude-code) skill that makes a coding agent's tool calls easy for a person to follow. The agent writes shell commands as commented narration with labeled output, and follows every tool call with a one-line execution summary saying what came back and what it means.

The person watching, or reading the session later, doesn't see the agent's reasoning and often skips raw output. With this skill they can follow the work through the comments and summaries alone.

## What it looks like

```bash
ssh -o BatchMode=yes build-host bash -s <<'EOF'
# The job's output folder and its log
JOB_DIR=/srv/jobs/nightly-2041
LOG="$JOB_DIR/run.log"

# result.json appears only when the job has finished
ls "$JOB_DIR/result.json" 2>/dev/null || echo "result.json: not yet"

# How far it has got
echo "tests passed: $(grep -c 'PASSED' "$LOG") of 120"
EOF
```

↳ **Execution summary** · still running, 84 of 120 tests passed, no result.json yet

The same kind of summary follows file edits, calls to other services and subagents:

↳ **Execution summary** · `cli/parser.py`: added a `--dry-run` flag to `main()`; the 12 parser tests pass

↳ **Execution summary** · created issue "Retry dropped uploads" (#482) in the team tracker

The full rules are in [SKILL.md](SKILL.md): how to write commands, how to shape their output, what each kind of summary says, a check to run before sending, and shell pitfalls that make output misleading.

## Install

### As a Claude Code plugin

In a Claude Code session, add this repo as a plugin marketplace, then install the plugin:

```
/plugin marketplace add BigRedT/readable-tool-calls
/plugin install readable-tool-calls@readable-tool-calls
```

The install command opens the plugin panel, where you choose a scope before it installs. From a shell, `claude plugin marketplace add BigRedT/readable-tool-calls` and `claude plugin install readable-tool-calls@readable-tool-calls` do the same, installing for your user by default.

### As a plain skill

For all your projects, clone it into your user-level skills folder, and pull to update:

```bash
git clone https://github.com/BigRedT/readable-tool-calls ~/.claude/skills/readable-tool-calls
git -C ~/.claude/skills/readable-tool-calls pull
```

For a single repository, copy `SKILL.md` to `.claude/skills/readable-tool-calls/SKILL.md` inside it.

Use one route, not both: two copies would load the same skill twice.

## Make it always on

Claude Code loads a skill when its description matches the task, so in a long session an agent can drift and stop applying it. To make it apply to every tool call, add this to `~/.claude/CLAUDE.md`, which loads at the start of every session:

```markdown
## Tool calls
- Follow the `readable-tool-calls` skill for every tool call, all session: shell commands as commented steps with labeled output, and after every tool call a plain line `↳ **Execution summary** · <what came back and what it means>` (reads and searches may share one per batch). Load the skill before the first tool call of a session.
```

## License

[MIT](LICENSE)
