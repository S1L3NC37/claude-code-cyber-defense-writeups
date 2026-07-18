# Building Defensive Tooling with Claude Code: A Sysmon Parser

*Building a Python parser for Sysmon process-creation events, and what the AI-assisted workflow does and doesn't change about defensive work.*

---

## What I built

A command-line Python tool that parses Sysmon Event ID 1 (process creation) records out of XML, extracts the fields that matter for detection, and filters on them.

It handles:

- **Single events and multi-event files**, with a clear error on malformed input instead of silent failure
- **Namespaced XML**, which is the trap that would have quietly broken everything (more on that below)
- **Filtering** by process, user, and integrity level, with case normalization and correct exit codes on bad values
- **Substring matching within a field** for hunting encoded PowerShell, matching either `encoded` or the `-enc` flag
- **Three output formats** (JSON, JSONL, CSV), CSV with correct Windows line endings
- **A `--stats` summary**, verified against a generated 5,000-event file

The parser is a normal Python script. It runs anywhere Python runs and doesn't require Claude Code to be open or installed.

I built it while working through *AI Cyber Defense Ops*, a course on using the Claude ecosystem for blue-team work. The parser solves a problem that's been solved many times over. I built it anyway, because the goal was to run the full workflow once and see where it holds up, not to ship something novel.

---

## The part I'd want a defender to read: the namespace trap

The first version worked on the first sample immediately.

[![First parser run: clean JSON output, plus the explanation of the namespace trap](https://github.com/S1L3NC37/claude-code-cyber-defense-writeups/raw/main/images/04-first-parser-output.png)](/S1L3NC37/claude-code-cyber-defense-writeups/blob/main/images/04-first-parser-output.png)

The thing worth pointing out is what I would have walked straight into writing this by hand. Sysmon XML puts every element inside a namespace. Search for a field name the obvious way and you get nothing back — no error, no crash, just empty fields and JSON full of nulls. A parser that fails loudly is an inconvenience. A parser that returns well-formed JSON with silently empty fields is a detection gap you don't find until you need the data.

The tool handled it and explained why. That's the class of problem I'd have burned real time on staring at blank output.

---

## Sample data before code

Before writing the parser, I built the test cases: three Sysmon events — one benign, one noisier, one carrying a suspicious encoded PowerShell command.

[![Three sample Sysmon fixtures with realistic fields and a decoded payload](https://github.com/S1L3NC37/claude-code-cyber-defense-writeups/raw/main/images/03-three-sample-fixtures.png)](/S1L3NC37/claude-code-cyber-defense-writeups/blob/main/images/03-three-sample-fixtures.png)

The suspicious sample carries a base64 blob that decodes to a real download command with a MITRE technique tag, which is the kind of thing you'd actually want a detection to catch. Writing fixtures first means you're building against something concrete instead of validating after the fact — a habit worth keeping regardless of tooling.

---

## Starting the project

Running `/init` in an empty folder taught me something immediately: it refused to generate a project context file for an empty directory.

[![/init refusing to invent a CLAUDE.md for an empty directory](https://github.com/S1L3NC37/claude-code-cyber-defense-writeups/raw/main/images/01-init-empty-project-refusal.png)](/S1L3NC37/claude-code-cyber-defense-writeups/blob/main/images/01-init-empty-project-refusal.png)

The command documents what exists; it doesn't invent an architecture. It asked what I was building instead. Once I described the plan, it wrote a real context file that every future session loads automatically, and flagged itself as a starting document to update as actual code appears.

[![The initial CLAUDE.md, written after I described the project](https://github.com/S1L3NC37/claude-code-cyber-defense-writeups/raw/main/images/02-initial-claude-md.png)](/S1L3NC37/claude-code-cyber-defense-writeups/blob/main/images/02-initial-claude-md.png)

---

## The break that was supposed to happen

The parser worked on one event. Pointed at a file with several, it broke — which the course said it would.

[![Root detection checked across single-event, multi-event, and bad-root inputs](https://github.com/S1L3NC37/claude-code-cyber-defense-writeups/raw/main/images/05-root-detection-verified.png)](/S1L3NC37/claude-code-cyber-defense-writeups/blob/main/images/05-root-detection-verified.png)

The fix taught it to handle both a bare event and a wrapper of many, and to error clearly on anything else. The part I'm glad I did was re-checking that the single-event case still passed afterward. Fixing the new thing and quietly breaking the old thing is easy, and regression only gets caught if you go looking.

---

## Planning before typing

For a larger change — filtering by process, user, and integrity level — I used plan mode instead of diving in.

[![Plan mode: the filtering plan proposed and approved before any code](https://github.com/S1L3NC37/claude-code-cyber-defense-writeups/raw/main/images/06-plan-mode-approved.png)](/S1L3NC37/claude-code-cyber-defense-writeups/blob/main/images/06-plan-mode-approved.png)

Where I landed on when it's worth it: if there's a real decision with trade-offs, plan first. The earlier fixes were obvious, so I just asked. This one had actual choices about how filtering composed and what the output looked like.

The planning paid off at verification. The filtering ran against a full set of cases including the annoying ones — a bad integrity value rejected with the right exit code, a lowercase value normalized.

[![The verification cases passing, edge cases and all](https://github.com/S1L3NC37/claude-code-cyber-defense-writeups/raw/main/images/07-verification-suite-passing.png)](/S1L3NC37/claude-code-cyber-defense-writeups/blob/main/images/07-verification-suite-passing.png)

That table is the argument for plan mode. The carefulness was designed in rather than bolted on.

---

## Going past the exercise: hunting encoded PowerShell

Filtering was the course's feature. This next piece was mine, and it's where the exercise turned into detection work.

I wanted to catch encoded PowerShell by matching either the word `encoded` or the `-enc` flag — a different shape than the other filters, an OR *within* one field rather than an AND across fields.

[![Working through the encoded-or-enc case and a command-line gotcha](https://github.com/S1L3NC37/claude-code-cyber-defense-writeups/raw/main/images/08-encoded-enc-gotcha.png)](/S1L3NC37/claude-code-cyber-defense-writeups/blob/main/images/08-encoded-enc-gotcha.png)

Two things came out of it. Encoded PowerShell is a real technique and "match any of these" is how you'd actually hunt it, so this was the first piece driven by detection reasoning rather than the lesson plan. And it surfaced a practical gotcha: a filter value starting with a dash, like `-enc`, gets eaten as an option unless you pass it a specific way. That's the kind of thing you only hit by running your own case instead of the tutorial's.

---

## Output formats and scale

Last, I stretched the tool: JSON, JSONL, and CSV output, each verified against the multi-event file.

[![Three output formats checked, including a Windows CSV line-ending fix](https://github.com/S1L3NC37/claude-code-cyber-defense-writeups/raw/main/images/11-format-verification.png)](/S1L3NC37/claude-code-cyber-defense-writeups/blob/main/images/11-format-verification.png)

The CSV path handled a Windows line-ending problem I wouldn't have known to check for. Then a `--stats` summary run against a generated 5,000-event file, pushed to the background since it ran long enough to matter.

[![The stats run pushed to the background over a 5,000-event file](https://github.com/S1L3NC37/claude-code-cyber-defense-writeups/raw/main/images/12-stats-backgrounded.png)](/S1L3NC37/claude-code-cyber-defense-writeups/blob/main/images/12-stats-backgrounded.png)

Worth noting: backgrounding only does anything when a task actually runs long enough to catch. On three-event fixtures it's a no-op. On bulk logs it matters.

---

## Saving state

Version control first — a local git repo with a real first commit, nothing pushed anywhere. A save point and a record of decisions on my own machine.

[![The first git commit, a local repo with a descriptive message](https://github.com/S1L3NC37/claude-code-cyber-defense-writeups/raw/main/images/09-first-git-commit.png)](/S1L3NC37/claude-code-cyber-defense-writeups/blob/main/images/09-first-git-commit.png)

*(Name and email in the commit are boxed out; they came from local context.)*

Then a handoff document capturing what got built, how to use it, what's left, and why the main decisions went the way they did.

[![A HANDOFF.md capturing state for a future session](https://github.com/S1L3NC37/claude-code-cyber-defense-writeups/raw/main/images/10-handoff-created.png)](/S1L3NC37/claude-code-cyber-defense-writeups/blob/main/images/10-handoff-created.png)

I kept these working docs local rather than committing them, which was its own small lesson in being deliberate about what belongs in a repo.

---

## What actually transfers

**The workflow generalizes. The judgment doesn't.** Context file first, fixtures before code, small steps, plan the big changes, save state — none of that is about Sysmon or XML. Swap the log type and every step still applies. But knowing which fields matter for which attack, what a realistic malicious sample looks like, and when output is quietly wrong stays with me. The workflow makes it faster to apply what I already know; it doesn't supply it.

**It's a helper, not an answer box.** The intentional break, the regression check, the encoded-PowerShell detour — all of it kept the decisions with me and used the model to move through the mechanical parts. That's the difference between using this to skip thinking and using it to think faster.

**The output outlives the session.** The parser is a plain Python script. It runs without Claude Code installed. It's not a toy that only works while a chat window is open.

---

## Limits

This made me faster, not more expert. The detection judgment that decides whether a result is any good didn't come from this exercise and can't be handed off. The loop multiplies existing knowledge rather than replacing it.

And the parser is a learning project, not production tooling. No test suite, no packaging, one event type, XML only. Those are written down as future work rather than hidden.

I came into this new to the Claude ecosystem and asked a lot of basic questions along the way — what an EVTX file is, whether the tool would run outside Claude Code, whether a keyboard shortcut was mine to press or the model's to handle. The full unedited version of those questions is part of how I learned this, and I'd rather document the actual path than a cleaned-up one.

---

## Future work

- Test suite and packaging
- Additional Sysmon event types beyond ID 1
- Native EVTX input rather than pre-converted XML
- Field mapping to ATT&CK techniques in the output
