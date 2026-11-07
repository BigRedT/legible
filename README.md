# readable-tool-calls

**Follow what your coding agent is doing without reading raw output.**

![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)
![Claude Code skill and plugin](https://img.shields.io/badge/Claude%20Code-skill%20%26%20plugin-D97757.svg)

When a coding agent works, it runs dozens of commands and edits, calls other services and hands work to subagents. You see a stream of dense one-liners and unlabeled output, and the agent rarely says what each result meant. This [Claude Code](https://claude.com/claude-code) skill makes the agent narrate its work: commands read like a commented script, output explains itself, and every tool call ends with a one-line summary of what came back.

## Before and after

The agent checks on a nightly test job running on another machine.

**Without the skill**

```bash
ssh build-host 'pgrep -f "nightly_build.*2041" >/dev/null && echo running || echo not running; ls /srv/jobs/nightly-2041; grep -c PASSED /srv/jobs/nightly-2041/run.log'
```

```
running
run.log
84
```

Is 84 good? Did the job finish? You have to decode the command to find out, and the agent has already moved on.

**With the skill**

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

```
process: running
result.json: not yet
tests passed: 84 of 120
```

↳ **Execution summary** · still running, 84 of 120 tests passed, no result.json yet

You can read the comments alone and know the plan, read the output alone and know the state, or read just the summary.

## What it does

- **Commands read as narration.** Every step gets a comment, one step per line, and remote work goes in a heredoc script instead of a quoted one-liner.
- **Output explains itself.** Values are labeled, logs are filtered to the lines that matter, both outcomes of a check are stated, and secrets are never printed.
- **Every tool call ends with a summary.** Shell commands, file edits, calls to other services and subagents each get one line saying what came back and what it means. A batch of reads shares one.
- **It holds up in long sessions.** A short check before each reply catches dropped summaries, the usual way agents drift.
- **It knows the shell traps.** zsh quirks, heredocs that silently swallow the rest of a script, and `grep` counts that mislead.

## A session at a glance

With the skill, a stretch of work reads like a log you can scan:

↳ **Execution summary** · read the three config loaders: only `load_env()` honors overrides, so the bug is in `load_file()`

↳ **Execution summary** · `cli/parser.py`: added a `--dry-run` flag to `main()`; the 12 parser tests pass

↳ **Execution summary** · full test suite: 1,204 passed, 3 skipped, none failed

↳ **Execution summary** · created issue "Retry dropped uploads" (#482) in the team tracker, assigned to nobody yet

The full rules are in [SKILL.md](SKILL.md).

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

Raise `version` in `.claude-plugin/plugin.json` with each change to `SKILL.md`, so plugin installs pick up the new copy.

## License

[MIT](LICENSE)
