# readable-tool-calls

A [Claude Code](https://claude.com/claude-code) skill that makes a coding agent's tool calls easy for a person to follow. The agent writes shell commands as commented narration with labeled output, and follows every tool call with a one-line execution summary saying what came back and what it means.

The person watching, or reading the session later, doesn't see the agent's reasoning and often skips raw output. With this skill they can follow the work through the comments and summaries alone.

## What it looks like

A status check on a remote job, written as the skill asks:

```bash
ssh -o BatchMode=yes build-host bash -s <<'EOF'
# The job's output folder and its log
JOB_DIR=/srv/jobs/nightly-2041
LOG="$JOB_DIR/run.log"

# Is the build process still alive?
if pgrep -f "nightly_build.*2041" > /dev/null; then
  echo "process: running"
else
  echo "process: not running"
fi

# result.json appears only when the job has finished
if [ -f "$JOB_DIR/result.json" ]; then
  echo "result.json: present"
else
  echo "result.json: not yet"
fi

# How far it has got: count the lines that start with PASSED
if [ -f "$LOG" ]; then
  echo "tests passed: $(grep -c '^PASSED' "$LOG") of 120"
else
  echo "run.log: not created yet"
fi
EOF
```

↳ **Execution summary** · still running, 84 of 120 tests passed, no result.json yet

The same kind of summary follows file edits, calls to other services and subagents:

↳ **Execution summary** · `cli/parser.py`: added a `--dry-run` flag to `main()`; the 12 parser tests pass

↳ **Execution summary** · created issue "Retry dropped uploads" (#482) in the team tracker, assigned to nobody yet

The full rules are in [SKILL.md](SKILL.md): how to write commands, how to shape their output, what each kind of summary says, a check to run before sending, and shell pitfalls that make output misleading.

## Install

Pick one route. With both, two copies of the skill load (the plugin's is named `readable-tool-calls:readable-tool-calls`), and they can drift apart.

### As a Claude Code plugin

In a Claude Code session, add this repo as a plugin marketplace, then install the plugin:

```
/plugin marketplace add BigRedT/readable-tool-calls
/plugin install readable-tool-calls@readable-tool-calls
```

The install command opens the plugin panel, where you choose a scope before it installs. From a shell, `claude plugin marketplace add BigRedT/readable-tool-calls` and `claude plugin install readable-tool-calls@readable-tool-calls` do the same, installing for your user by default.

To update, refresh the marketplace and then the plugin:

```bash
claude plugin marketplace update readable-tool-calls
claude plugin update readable-tool-calls@readable-tool-calls
```

### As a plain skill

For all your projects, clone it into your user-level skills folder, and pull to update:

```bash
git clone https://github.com/BigRedT/readable-tool-calls ~/.claude/skills/readable-tool-calls
git -C ~/.claude/skills/readable-tool-calls pull
```

For a single repository, copy `SKILL.md` to `.claude/skills/readable-tool-calls/SKILL.md` inside it.

## Make it always on

Claude Code loads a skill when its description matches the task, so in a long session an agent can drift and stop applying it. To make it apply to every tool call, add a pointer to `~/.claude/CLAUDE.md`, which loads at the start of every session:

```markdown
## Tool calls
- Follow the readable-tool-calls skill for every tool call, all session (installed as a plugin it is named readable-tool-calls:readable-tool-calls). Load it before the first tool call; it ends each tool call with a plain line `↳ **Execution summary** · <what came back and what it means>`.
```

The pointer names the skill and its one-line format; the full rules stay in the skill, so updating the skill updates them.

## Changing the skill

Raise `version` in `.claude-plugin/plugin.json` with each change, so plugin installs pick up the new copy.

## License

[MIT](LICENSE)
