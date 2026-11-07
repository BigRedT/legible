---
name: readable-tool-calls
description: How a coding agent makes its tool calls easy for a human to follow. Shell commands are written as commented narration with labeled, filtered output, and every tool call (or batch of reads) is followed by a "↳ Execution summary" line saying what came back, what changed and what it means. Use this for every shell command, file edit or write, external service or MCP call, web fetch, subagent, and batch of file reads or searches, for the whole session, including short status checks and routine steps, even when nobody has mentioned it.
---

# Readable tool calls

A person follows a coding agent's work through its tool calls and what comes back. They don't see the agent's reasoning, they often skip raw output, and they may read the session later. So shell commands should read as narration, their output should make sense on its own, and every tool call should end in a plain-language summary.

Apply all of this for the whole session. The usual failure is drift: summaries get dropped on quick status checks, on background jobs, on calls to other services, and late in long sessions, exactly when the person relies on them most.

## 1. Write shell commands as narration

- **Comment every step.** A short `#` line above each step says what it does and why. Someone reading only the comments should understand the plan. Comment the steps inside heredocs and inline scripts too.
- **One step per line.** Avoid long `&&` or `;` chains. Use `&&` only when a later step must not run after a failure, and break the line there too.
- **Write if/else as `if … then … else … fi`.** The shortcut `A && B || C` also runs `C` when `B` fails, so it can print both branches.
- **Split long argument lists** with `\`, one flag (or flag and value) per line.
- **Name things once.** Put repeated paths, job IDs and hosts in variables at the top (`JOB_DIR=…`), not inline over and over.
- **Use a heredoc script for remote or multi-step work**, never a quoted one-liner with escaped quotes: `ssh host bash -s <<'EOF'`. Quote the delimiter so variables expand on the remote side, and nest a different one (`<<'PY'`) for an inner script.
- **Make waiting loops readable and bounded.** Give them a maximum number of iterations, a comment saying what ends the loop, and a final labeled line with the outcome and the time.
- **One purpose per command.** If a command does several unrelated things, split it or head each part with a comment.
- **Describe it plainly.** If your tool asks for a short description of each command, say in plain words what it does ("Check whether the nightly build finished"). Don't repeat the command text.

## 2. Make command output readable on its own

- **Label what you print.** Write `echo "tests passed: $n of 120"`, not a bare number. Head each section of output (`echo "----- last 20 log lines"`).
- **Print the answer, not the haystack.** Show counts, states and the few lines that matter, filtered with `grep` and capped with `head` or `tail`. From JSON, print the fields you need by name.
- **Label both outcomes of a check.** Print "result.json: present" or "result.json: not yet", not a bare path in one case and a message in the other. A missing file or empty log should read as a state, not as a blank.
- **Never print secrets.** For keys, tokens, passwords and credential files, print the name, whether it is set and its length, never the value. Don't print a digest of a short secret such as a password or PIN, because it can be guessed back from the digest.

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

## 5. Shell pitfalls that make output misleading

Check which shell and system run your commands. These quirks produce results that look right and aren't:

- **zsh (the macOS default):** `status` is read-only, so a loop that assigns it dies at once. Use `state` or `rc`.
- **zsh:** an unquoted `$var` is not split into words, so a variable holding several paths arrives as one argument. Use an array (`files=(a b)` and then `"${files[@]}"`) or list the items.
- **Any shell:** in a heredoc script, a command that reads input, such as `ssh`, `docker exec -i`, `read` or `python3` with no script, swallows the rest of the script. The later steps silently never run, and the output looks complete. Give such a command `< /dev/null`.
- **Any shell:** `pgrep -f "<pattern>"` can match the shell running your own check when the pattern is on a command line, as in a quoted `ssh host '…'` or `bash -c` one-liner. Bracket one letter (`pgrep -f "[n]ightly_build"`) there. A heredoc script is not on any command line, so it can't match itself.
- **Any shell:** `grep -c error` also counts harmless warnings that contain the word ("…can yield errors"). Match what a real failure prints, such as `Traceback`, or anchor the pattern (`^PASSED`).
- **Any shell:** a command that fails before printing anything looks like an empty result once its output is piped into `grep` ("nothing found"). Check the exit code or show stderr before trusting an empty match. On macOS this happens with `timeout`, a GNU tool macOS lacks: use your tool's own time limit, or `gtimeout` from GNU coreutils.

## Examples

A shell command: commented steps, named paths, both outcomes of every check labeled, then the summary.

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

The same check written the hard-to-read way:

```bash
ssh build-host 'pgrep -f "nightly_build.*2041" >/dev/null && echo running || echo not running; ls /srv/jobs/nightly-2041; grep -c PASSED /srv/jobs/nightly-2041/run.log'
```

Summaries after other kinds of call:

↳ **Execution summary** · `cli/parser.py`: added a `--dry-run` flag to `main()`; the 12 parser tests pass

↳ **Execution summary** · created issue "Retry dropped uploads" (#482) in the team tracker, assigned to nobody yet

↳ **Execution summary** · read the three config loaders: only `load_env()` honors overrides, so the bug is in `load_file()`
