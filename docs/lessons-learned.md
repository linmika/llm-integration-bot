# Lessons learned

From operating the bot and from a structured review of a "v2" test copy that
added an insight-memory feature.

## The insights sheet detour

**What we built:** 
a `/save` command. After a thread was resolved, an internal user typed `/save`; 
the agent extracted `category | symptom keywords | error code |
endpoint | root cause | fix | doc link` from the thread and appended a row to a
spreadsheet. Before answering any new error question, the agent fetched the *entire*
sheet and scanned it for a match.

**What went wrong:**

- The platform already had a native "learn this message into the learnt KB"
  command, with semantic retrieval and citations. We rebuilt a worse version of it.
- Full-table read on every turn — latency grew with every row saved.
- No dedup: the same fix got saved twice on consecutive days.
- No guard for a bad `/save`: one row had every field set to `Unknown` because
  someone typed `/save` with no thread above it. The prompt said "don't save in that
  case"; the model didn't comply reliably.
- "Reply `Saved ✓` after appending" had no branch for a failed write, so the bot
  could claim success without checking the tool result.

**What to do instead:** 
write to the learnt KB through the platform's native
mechanism; if you *must* keep a table, read it once into the KB, not on every turn.
Put dedup and "nothing to save" checks in **code**, not in the prompt.

## Channel safety

Internal-only behaviours (thread parsing, `/save`) ended up in the same prompt that
served external partners, because the "v2" copy was forked from the shared base.
Nothing gated `/save` by channel. A partner on the public messenger could, in
principle, write into our internal memory.

**Rule:** anything that mutates state lives only in the internal multi-agent, and
the external bot's prompt should never know those commands exist. Fork prompts
*per channel*, not per feature.

## Prompt hygiene

Things a review found in a prompt that had been edited by several people over a
year:

- Two different "check X first" instructions that contradicted each other
  (learnt KB first vs. sheet first). Pick one ordering and state it once.
- "Do not use other links" repeated three times. Repetition doesn't improve
  compliance; it just makes the prompt harder to maintain.
- A redaction rule ("never write credentials") that applied to the save path but
  not to the escalation-summary path — the one place partners most often paste
  secrets.
- Skills numbered 0–6 in insertion order rather than execution order. Renumber when
  you restructure.

## Platform toggles that were off and shouldn't have been

| Toggle | Why it matters |
|--------|----------------|
| Citations | The whole prompt fights hallucinated links; citations let users verify. |
| Early response ("I'm looking into this…") | Hides tool/retrieval latency on both channels. |
| Collect feedback in internal chat | Free thumbs-up/down signal. |
| Explicit reasoning effort / verbosity on the frontier model | Unset meant "whatever the default is this week". |

Cheap to flip; nobody owned the checkbox.

## Secrets in agent code

Rule-driven agents are usually generated from a step-by-step spec, and the spec is
where people paste tokens. The literal then lands in the generated Python and from
there in every config export, review transcript and debug log.

**Rule:** credentials go in the platform's credential store / environment scope and
are referenced by name. Rotate anything that has ever been pasted into a prompt.

## Ownership drift

A bot that lives a year gets created by one person, edited by another and forked
by a third. Keep a one-line changelog in the agent description ("Removed X → Y
pipe; adjusted prompt for Z") so the next editor can reconstruct why it looks the
way it does.

## Review checklist

Before promoting any prompt or agent change:

- [ ] Does the external bot's prompt contain anything only internal users should
      trigger?
- [ ] Is every "check A before B" ordering stated exactly once?
- [ ] Does every tool call that writes have a failure branch?
- [ ] Does every wait-for-user have a timeout and a recovery message?
- [ ] Are redaction rules applied on *every* output path (answer, save, escalate)?
- [ ] Any secret as a string literal?
- [ ] Is the change noted in the agent description?
- [ ] Have the logs from the last N real conversations been read, not just test
      prompts?
