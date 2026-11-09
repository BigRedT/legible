# Legible

**Make your agent's commands human readable and get an execution summary after every tool call.**

![Claude Code skill and plugin](https://img.shields.io/badge/Claude%20Code-skill%20%26%20plugin-D97757.svg)
![Codex skill and plugin](https://img.shields.io/badge/Codex-skill%20%26%20plugin-10A37F.svg)
![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)

When a coding agent works, it runs dozens of commands and edits, calls other services and hands work to subagents. You see a stream of dense one-liners and unlabeled output, and the agent rarely says what each result meant. This skill, for [Claude Code](https://claude.com/claude-code) and [Codex](https://developers.openai.com/codex), makes the agent narrate its work: commands read like a commented script, output explains itself, and every tool call ends with a one-line summary of what came back.

## Before and after

In one command, the agent copies training code to a GPU machine, starts the run and checks on it.

![The same command without and with Legible, side by side: a dense one-liner with unlabeled output, then a commented script with labeled output and an execution summary, with notes marking each difference](assets/before-after.png)

With Legible, you can read the comments alone and know the plan, read the output alone and know the state, or read just the summary.

<details>
<summary>The same example as text</summary>

**Without Legible**

```bash
rsync -az --exclude .git --exclude .venv --exclude checkpoints ./ gpu-box:sft/ && ssh gpu-box 'cd sft; nohup torchrun --nproc_per_node=8 train.py > train.log 2>&1 < /dev/null & echo $!; sleep 90; grep -c Traceback train.log; tail -1 train.log'
```

```
48213
0
step 12 | loss 2.39 | 3.4 s/step
```

What is 48213? Is 0 good or bad? You have to decode the command to find out, and the agent has already moved on.

**With Legible**

```bash
# ═════════════════════════════════════════════════════════════════
# Goal: start training on the GPU machine and check it runs cleanly
# ═════════════════════════════════════════════════════════════════

# The GPU machine
HOST=gpu-box

# 1. Copy the code over, skipping git history, venv and checkpoints
rsync -az --exclude .git --exclude .venv --exclude checkpoints \
  ./ "$HOST:sft/" &&

# If the copy worked, run the rest on the GPU machine (up to EOF)
ssh "$HOST" bash -s <<'EOF'
# 2. Train on all 8 GPUs, detached so it keeps going after ssh exits
cd sft
nohup torchrun --nproc_per_node=8 train.py \
  > train.log 2>&1 < /dev/null &
echo "started: pid $!"

# 3. Loading the model takes a while, so wait 90 seconds, then check
sleep 90
echo "tracebacks in log: $(grep -c Traceback train.log)"
echo "last log line: $(tail -1 train.log)"
EOF
```

```
started: pid 48213
tracebacks in log: 0
last log line: step 12 | loss 2.39 | 3.4 s/step
```

↳ **Execution summary** · code synced and training started (pid 48213): after 90 s it's at step 12, loss 2.39 at 3.4 s per step, with no tracebacks in the log

</details>

## What it does

- **Commands read as narration.** A multi-step command opens with its goal, set off by rule lines; every step gets a comment, one step per line, and remote work goes in a heredoc script instead of a quoted one-liner.
- **Output explains itself.** Values are labeled, logs are filtered to the lines that matter, both outcomes of a check are stated, and secrets are never printed.
- **Every tool call ends with a summary.** Shell commands, file edits, calls to other services and subagents each get one line saying what came back and what it means. A batch of reads shares one.
- **It holds up in long sessions.** A short check before each reply catches dropped summaries, the usual way agents drift.
- **It changes presentation only.** The agent runs the same commands and makes the same edits; only how they are written and reported changes.

## A session at a glance

With the skill, a stretch of work reads like a log you can scan:

↳ **Execution summary** · read the three config loaders: only `load_env()` honors overrides, so the bug is in `load_file()`

↳ **Execution summary** · `cli/parser.py`: added a `--dry-run` flag to `main()`; the 12 parser tests pass

↳ **Execution summary** · full test suite: 1,204 passed, 3 skipped, none failed

↳ **Execution summary** · created issue "Retry dropped uploads" (#482) in the team tracker, assigned to nobody yet

The full rules are in [skills/legible/SKILL.md](skills/legible/SKILL.md). The root `SKILL.md` links to that same file for existing plain-skill installs.

## Install

### Claude Code

Pick one route. With both, two copies of the skill load (the plugin's is named `legible:legible`), and they can drift apart.

#### As a plugin

In a Claude Code session, add this repo as a plugin marketplace, then install the plugin:

```
/plugin marketplace add BigRedT/legible
/plugin install legible@legible
```

The install command opens the plugin panel, where you choose a scope before it installs. From a shell, `claude plugin marketplace add BigRedT/legible` and `claude plugin install legible@legible` do the same, installing for your user by default.

To update, refresh the marketplace and then the plugin:

```bash
# Refresh the marketplace, then update the installed plugin.
claude plugin marketplace update legible
claude plugin update legible@legible
```

#### As a plain skill

For all your projects, clone it into your user-level skills folder, and pull to update:

```bash
# Install the skill for your user.
mkdir -p ~/.claude/skills
git clone https://github.com/BigRedT/legible ~/.claude/skills/legible
# Pull subsequent updates.
git -C ~/.claude/skills/legible pull
```

For a single repository, copy `SKILL.md` to `.claude/skills/legible/SKILL.md` inside it.

### Codex

Pick either the plugin or the plain skill to avoid loading two copies.

#### As a plugin

From a shell with a Codex CLI that supports `codex plugin`, register this repository as a marketplace and install the plugin:

```bash
# Register the Git-backed marketplace.
codex plugin marketplace add BigRedT/legible
# Install and enable the skill plugin.
codex plugin add legible@legible
```

To update, refresh the marketplace and install its current plugin version:

```bash
# Fetch the latest marketplace snapshot.
codex plugin marketplace upgrade legible
# Refresh the installed plugin from that snapshot.
codex plugin add legible@legible
```

Check the installation with `codex plugin list --marketplace legible`, then start a new Codex session. Invoke the bundled skill as `$legible:legible`, or follow the always-on instructions below.

The package uses a portable `plugin.json`, the standard `skills/` directory, and a Codex marketplace at `.agents/plugins/marketplace.json`. See the [official plugin packaging documentation](https://developers.openai.com/plugins/build/plugins).

#### As a plain skill

Codex reads user-level skills from `~/.agents/skills`. Clone it there, and pull to update:

```bash
# Install the skill for your user.
mkdir -p ~/.agents/skills
git clone https://github.com/BigRedT/legible ~/.agents/skills/legible
# Pull subsequent updates.
git -C ~/.agents/skills/legible pull
```

For a single repository, copy `SKILL.md` to `.agents/skills/legible/SKILL.md` inside it.

### Both agents, one copy

If you use both, install the plain skill for Claude Code, then link Codex to the same folder, so one `git pull` updates both:

```bash
# Link the existing Claude Code plain-skill install into Codex's skill directory.
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/legible ~/.agents/skills/legible
```

### Migrating from readable-tool-calls

Legible was previously named `readable-tool-calls`. For plugin installs, uninstall `readable-tool-calls@readable-tool-calls` and remove its marketplace, then follow the install commands above for your agent. The new plugin and marketplace are both named `legible`.

For a plain-skill checkout, pull the latest version; its existing folder name can stay as-is. If you installed by copying `SKILL.md`, replace it with the new file. Update any always-on instruction pointers to `legible` (or `legible:legible` for a plugin install).

## Make it always on

Agents load a skill when its description matches the task, so in a long session an agent can drift and stop applying it. To make it apply to every tool call, add a pointer to the instructions file your agent reads at the start of every session: `~/.claude/CLAUDE.md` for Claude Code, `~/.codex/AGENTS.md` for Codex.

```markdown
## Tool calls
- Follow the legible skill for every tool call, all session (as a plugin it is named legible:legible). Load it before the first tool call; it ends each tool call with a plain line `↳ **Execution summary** · <what came back and what it means>`.
```

The pointer names the skill and its one-line format; the full rules stay in the skill, so updating the skill updates them.

## Changing the skill

Edit `skills/legible/SKILL.md`; the root `SKILL.md` is a relative symlink, so there is only one copy to maintain. The packaged skill must be a real file because plugin installers may skip symlinks when copying packages.

Raise `version` together in `plugin.json` and `.claude-plugin/plugin.json` whenever the skill or package changes, so both agents' plugin installs pick up the new version.

To test a local checkout with Codex, register its absolute path with `codex plugin marketplace add /absolute/path/to/legible`, then run the same plugin install and list commands above. Use this instead of Git registration for that marketplace name. Local marketplaces read the checkout directly; `marketplace upgrade` is for Git-backed marketplaces.

If you change the example, update `assets/before-after.html` to match and render the image again (macOS, with Chrome):

```bash
# Render the updated example image with Chrome.
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --hide-scrollbars \
  --force-device-scale-factor=2 --window-size=1200,936 \
  --screenshot=assets/before-after.png "file://$PWD/assets/before-after.html"
```

## License

[MIT](LICENSE)
