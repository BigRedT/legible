---
name: legible
description: How a coding agent presents its tool calls and their results so a human can follow the session without reading raw output. Shell commands are written as commented narration with labeled, filtered output, and every tool call (or batch of reads) is followed by a "↳ Execution summary" line saying what came back, what changed and what it means. It changes presentation only, not what the agent does. Use this for every shell command, file edit or write, external service or MCP call, web fetch, subagent, and batch of file reads or searches, for the whole session, including short status checks and routine steps, even when nobody has mentioned it.
---

# Legible

A person follows a coding agent's work through its tool calls and what comes back. They don't see the agent's reasoning, they often skip raw output, and they may read the session later. So shell commands should read as narration, their output should make sense on its own, and every tool call should end in a plain-language summary.

This skill changes how tool calls and results are presented, not what the agent does: run the same commands, edits and checks you would anyway, written and reported so a person can follow them.

Apply it for the whole session. The usual failure is drift: summaries get dropped on quick status checks, on background jobs, on calls to other services, and late in long sessions, exactly when the person relies on them most.

## 1. Write shell commands as narration

- **Comment every step.** A short `#` line above each step says what it does and why. Someone reading only the comments should understand the plan. Comment the steps inside heredocs and inline scripts too. Where shell syntax decides what runs, say it in words: "If the copy worked" for a trailing `&&`, "detached" for a trailing `&`.
- **Number the steps** when a command has three or more, so the comments read as a plan.
- **One step per line.** Avoid long `&&` or `;` chains on one line; break them across lines.
- **Break lines where the meaning changes**, with `\`: a tool's options, then what it acts on; a launcher's options, then the program it runs; a command, then where its output goes; one pipeline stage per line.
- **Name things once.** Put repeated paths, job IDs and hosts in variables at the top (`JOB_DIR=…`), not inline over and over.
- **Use a heredoc script for remote or multi-step work**, not a quoted one-liner with escaped quotes: `ssh host bash -s <<'EOF'`. Quote the delimiter so variables expand on the remote side, and nest a different one (`<<'PY'`) for an inner script. The comment above each heredoc says where its code runs and which marker ends it: "run the rest on the GPU machine (up to EOF)".
- **Make waiting loops readable.** Say in a comment what ends the loop, and finish with a labeled line giving the outcome and the time.
- **One purpose per command.** If a command does several unrelated things, split it or head each part with a comment.
- **Describe it plainly.** If your tool asks for a short description of each command, say in plain words what it does ("Copy the code to the GPU machine and start training"). Don't repeat the command text.

## 2. Make command output readable on its own

- **Label what you print.** Write `echo "tests passed: $n of 120"`, not a bare number. Head each section of output (`echo "----- last 20 log lines"`).
- **Print the answer, not the haystack.** Show counts, states and the few lines that matter, filtered with `grep` and capped with `head` or `tail`. From JSON, print the fields you need by name.
- **Label both outcomes of a check.** Print "result.json: present" or "result.json: not yet", not a bare path in one case and a message in the other.
- **Keep secrets out of the output.** For keys, tokens and passwords, print the name and whether it is set, not the value.

## 3. Report every tool call

Right after a tool call's result, before any other prose, write an execution summary:

↳ **Execution summary** · still running, 84 of 120 tests done, no failures so far

What it says depends on the kind of call:

- **Shell commands:** the number, state or error that decides the next step. "The command ran" says nothing.
- **File edits and writes:** what changed and where (file, function or section), and the test or format result if one ran. Several edits that make one change share one summary.
- **External services and MCP tools** (notes, chat, issue trackers, cloud APIs, web fetches): what was created, changed or found, with its title, link or ID. These results are the hardest for the person to see, and they change things outside the repo.
- **Subagents:** what the agent concluded, and what you have not verified yourself.
- **Reads and searches:** one summary per batch, saying what you learned that matters ("read the three test helpers: only `make_client()` sets a timeout"). Don't write one after every file read; that buries the summaries that matter.

Rules for every summary:

- **Every tool call is covered,** alone or in a batch. That includes routine checks, calls that return nothing ("no output; the file was written") and calls that fail. A missing summary leaves the person guessing whether the step worked.
- **Report only what the call showed.** Don't claim a job is running if the command never checked; say what is unknown instead.
- **Failures:** name what failed, quote the key error in a few words, and say what happens next if that isn't obvious.
- **Background work:** say what started, where its output goes, and how you will know it finished. When it finishes, give that result its own summary.
- **Several calls in one turn:** write one summary per call, in order, or one summary that names each.
- **Keep it to one or two lines.** Detail goes in the prose after it.
- **Keep the exact form:** a plain line of its own, with a blank line before and after, starting with `↳`, then bold **Execution summary**, then a middle dot. No blockquote and no code block. A fixed form lets the reader scan a long session for results.

## 4. Check before you send

Before ending any message that follows tool results, check:

1. Does every tool call since my last message have an execution summary (reads and searches may share one)?
2. Does every shell step carry a comment?
3. Did any remote or multi-step command go out as a quoted one-liner? If so, use a heredoc next time.

If you find you dropped summaries earlier, say so, add the missing ones briefly, and carry on.

## Examples

A shell command: numbered and commented steps, the host named once, lines broken where the meaning changes, a heredoc whose comment says where it runs, labeled output, then the summary.

```bash
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

↳ **Execution summary** · code synced and training started (pid 48213): after 90 s it's at step 12, loss 2.39 at 3.4 s per step, with no tracebacks in the log

The same command written the hard-to-read way:

```bash
rsync -az --exclude .git --exclude .venv --exclude checkpoints ./ gpu-box:sft/ && ssh gpu-box 'cd sft; nohup torchrun --nproc_per_node=8 train.py > train.log 2>&1 < /dev/null & echo $!; sleep 90; grep -c Traceback train.log; tail -1 train.log'
```

Summaries after other kinds of call:

↳ **Execution summary** · `cli/parser.py`: added a `--dry-run` flag to `main()`; the 12 parser tests pass

↳ **Execution summary** · created issue "Retry dropped uploads" (#482) in the team tracker, assigned to nobody yet

↳ **Execution summary** · read the three config loaders: only `load_env()` honors overrides, so the bug is in `load_file()`
