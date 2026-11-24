---
name: ask-all-subs
description: Use when Mason asks every provider a question or wants each model's opinion on something.
---

# Ask all subs

One question goes to one agent per provider. The thread that runs this skill
is the orchestrator: it sends the question out, reads every report, and answers
Mason here with the comparison.

## 1. Check the thread and ask Mason

Call `orchestrator_capabilities` once. Its `inheritedProviderInstanceId` is the
provider this thread runs on.

Then ask Mason with `AskUserQuestion`, in one call:

- **Ping** (every time, even if he picked one last time): iMessage (the T3
  iMessage line), iMessage through Hermes, or no ping, just reply in this thread.
- **Claude account** (only if this thread isn't on `claudeAgent` or
  `claudeAgent_charlie_claude`): Claude or Charlie Claude.

## 2. Send the question out

One `delegate_task` per provider, all in the same turn so they run in parallel:

| Provider | `providerInstanceId` | Model |
|---|---|---|
| Claude | this thread's Claude account (see below) | `claude-opus-5-5` |
| Codex | `codex` | `gpt-6.2-sol` |
| Cursor | `cursor` | `grok-4.7` |
| Antigravity | `antigravity` | `gemini-4-neon` |

Send only one Claude agent. Both Claude accounts run the same models, so asking
both just repeats the answer. Use the account this thread is on, since it's the
one that has usage right now. If this thread isn't on Claude, use the account
Mason picked. If the Claude agent fails on a usage limit, retry it once on the
other Claude account.

If a provider or model is missing from `orchestrator_capabilities`, or a task
fails to start, skip it and say so in the final answer. Don't swap in another
provider.

Give every agent the same brief, word for word, so the answers compare
cleanly. Children get no history, so the brief must stand alone:

```
Answer this question on your own. Other agents are answering it separately.

Question: <Mason's question, plus any context or files it needs>

Reply with:
1. Answer: your direct answer in a few sentences.
2. Reasoning: the key points or evidence behind it. Cite sources or file paths
   if you used any.
3. Confidence: high, medium, or low, and why.
4. Unsure about: anything you couldn't check or might have wrong.

Don't edit files or make changes. This is research only.
```

## 3. Wait for the reports

Follow each task's `next` until it finishes. Don't poll with `task_status`,
sleep, or watchers. If one agent stalls long after the others are done, cancel
it with `task_cancel` and note that it timed out.

## 4. Compare the reports

Read every report in full. Then write the answer for Mason:

```
## <question, short>

**Bottom line:** <the answer most agents support, or "split" if there isn't one>

**Agree on**
- <point> (Claude, Codex, Cursor)

**Disagree on**
- <topic>: Claude says X; Codex says Y. <which looks right and why, if you can tell>

**Only one raised**
- <point> (Antigravity)

**Confidence:** <each agent's stated confidence, one line>

**Missing:** <agents that were skipped or timed out, or "none">

**My take:** <your own recommendation after reading them all>
```

Keep it tight. Name which agent said what. If you can check a disputed fact
yourself (run a command, read a file, look it up), do it and say what you
found. Don't paste the full reports unless Mason asks for them.

## 5. Ping, then reply

If Mason chose a ping, send a one-line summary (the bottom line plus "full
comparison is in the thread"):

```bash
# iMessage (T3 line)
printf '%s' "<summary>" | ssh vps-mason '~/.local/bin/t3-imessage-say -'

# iMessage through Hermes
printf '%s' "<summary>" | ssh vps-mason '~/.local/bin/hermes send -t photon'
```

On the VPS itself, drop the `ssh vps-mason` part and run the command directly.
If the ping fails, try once more, then move on. Post the full comparison as
your reply in this thread either way.
