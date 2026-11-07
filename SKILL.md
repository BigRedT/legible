---
name: readable-tool-calls
description: How a coding agent presents its tool calls and their results so a human can follow the session without reading raw output. Shell commands are written as commented narration with labeled, filtered output, and every tool call (or batch of reads) is followed by a "↳ Execution summary" line saying what came back, what changed and what it means. It changes presentation only, not what the agent does. Use this for every shell command, file edit or write, external service or MCP call, web fetch, subagent, and batch of file reads or searches, for the whole session, including short status checks and routine steps, even when nobody has mentioned it.
---

# Readable tool calls

A person follows a coding agent's work through its tool calls and what comes back. They don't see the agent's reasoning, they often skip raw output, and they may read the session later. So shell commands should read as narration, their output should make sense on its own, and every tool call should end in a plain-language summary.

This skill changes how tool calls and results are presented, not what the agent does: run the same commands, edits and checks you would anyway, written and reported so a person can follow them.

Apply it for the whole session. The usual failure is drift: summaries get dropped on quick status checks, on background jobs, on calls to other services, and late in long sessions, exactly when the person relies on them most.

## 1. Write shell commands as narration

- **Comment every step.** A short `#` line above each step says what it does and why. Someone reading only the comments should understand the plan. Comment the steps inside heredocs and inline scripts too.
- **One step per line.** Avoid long `&&` or `;` chains on one line; break them across lines.
- **Split long argument lists** with `\`, one flag (or flag and value) per line.
- **Name things once.** Put repeated paths, job IDs and hosts in variables at the top (`JOB_DIR=…`), not inline over and over.
- **Use a heredoc script for remote or multi-step work**, not a quoted one-liner with escaped quotes: `ssh host bash -s <<'EOF'`. Quote the delimiter so variables expand on the remote side, and nest a different one (`<<'PY'`) for an inner script.
- **Make waiting loops readable.** Say in a comment what ends the loop, and finish with a labeled line giving the outcome and the time.
- **One purpose per command.** If a command does several unrelated things, split it or head each part with a comment.
- **Describe it plainly.** If your tool asks for a short description of each command, say in plain words what it does ("Check whether the nightly build finished"). Don't repeat the command text.

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

A shell command: commented steps, named paths, labeled output, then the summary.

```bash
ssh -o BatchMode=yes build-host bash -s <<'EOF'
# The job's output folder and its log
JOB_DIR=/srv/jobs/nightly-2041
LOG="$JOB_DIR/run.log"

# Is the build process still alive?
pgrep -f "nightly_build.*2041" > /dev/null \
  && echo "process: running" \
  || echo "process: not running"

# result.json appears only when the job has finished
test -f "$JOB_DIR/result.json" \
  && echo "result.json: present" \
  || echo "result.json: not yet"

# How far it has got
echo "tests passed: $(grep -c '^PASSED' "$LOG") of 120"
EOF
```

↳ **Execution summary** · still running, 84 of 120 tests passed, no result.json yet

The same check written the hard-to-read way:

```bash
ssh build-host 'pgrep -f "nightly_build.*2041" >/dev/null && echo running || echo not running; ls /srv/jobs/nightly-2041; grep -c PASSED /srv/jobs/nightly-2041/run.log'
```

Summaries after other kinds of call:

↳ **Execution summary** · `cli/parser.py`: added a `--dry-run` flag to `main()`; the 12 parser tests pass

↳ **Execution summary** · created issue "Retry dropped uploads" (#482) in the team tracker, assigned to nobody yet

↳ **Execution summary** · read the three config loaders: only `load_env()` honors overrides, so the bug is in `load_file()`
