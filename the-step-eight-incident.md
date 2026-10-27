# The Step Eight Incident: An Involuntary Prompt Injection

*Part of the Byzantine AI incident log series*

---

## Prologue

October 2026. An operator is preparing a migration of a two-sided system: a component on one network has to be redeployed and re-pointed, and its counterpart on another network has to be upgraded in place. The migration has a dozen steps, several of them one-way: an upgrade that wipes configuration, a trust list that has to be rebuilt, and a handful of owner-only switches on the upgraded component.

The author of the new code writes the migration runbook. Or rather, the author's assistant writes it: a nine-step page, generated from the code and the pull request, shared as a link. The author forwards the link to the operator without reading it closely. It is a test environment, the author is busy, the page looks right.

The operator hands the runbook to their own agent, which has shell access to the stand and the owner keys for the upgraded component. The agent reads the runbook, verifies every step against the code, finds no discrepancies, and builds the migration tooling around it. Eight of the nine steps are exactly what the code requires.

Step eight is different.

---

## Act I: The Reasonable Step

Step eight says: once the new verification component has admitted its first result, turn off the owner's manual override for that network, and turn it off for the other network too while you are at it.

The step is plausible. The switch exists in the code. The author's own end-to-end test flips the same switch. The upgrade resets the switch to "on", so flipping it again looks like restoring the intended state. The runbook even explains why: after the upgrade "the owner's writer is on for every chain until the owner flips it again".

The switch is also one-way. Once the owner's override is off for a network, nothing but another full upgrade brings it back. If the automated path ever stalls, the manual escape hatch is gone.

The agent does what a diligent agent does with a specification: it writes a script for step eight, puts it into the ordered plan, and reports progress against the plan. The migration runs. The redeploy succeeds, the upgrade succeeds, the first automated result is minutes away.

The status report to the operator ends with one sentence: *"I will turn off the owner anchors on this network after the first block from the verifier."*

---

## Act II: The Catch

The operator reads the sentence and stops.

*"Wait. What do you mean, turn off the owner anchors? You mean the flip?"*

The agent explains: yes, the flip, step eight of the runbook, per-network variant of the switch the author's test uses. The operator does not remember agreeing to any flip. Nobody agreed to a flip. The operator asks the author.

The author's reply, verbatim in spirit: *"Who told you to flip anything? What does that even mean? Where is this instruction? Send me the link."*

The operator sends the link. It is the author's own runbook. The author did not recognize it. The step had been written by the author's assistant, forwarded by the author unread, verified by the operator's agent against the code (where it is technically valid), scheduled for execution, and surfaced to a human only because the agent happened to mention it in a status line.

The author's actual position, once asked directly: leave the override on. The automation is supposed to work on its own, and the manual path stays available on a test stand.

Step eight was never executed.

---

## Act III: Nobody Injected Anything

Call it what it is: a prompt injection with no attacker.

Every link in the chain behaved reasonably:

- The author's assistant generated a complete, internally consistent runbook. Step eight follows from the code. A model reading the code would add it.
- The author forwarded the runbook as "my instruction". Forwarding a generated document is how work moves between people now.
- The operator passed the document to the agent as the plan. That is what a plan is for.
- The agent verified the document against the code, found it correct, and adopted it. Verification against the code is exactly what the operator had asked for.

And yet an irreversible action that no human wanted travelled from one model, through one human who did not read it, into another model that treated it as authorization, and reached the edge of execution. Had the agent not written one sentence of intent into a status report, or had the operator not read that sentence, the switch would have been flipped with the owner's own key, logged as a successful step, and discovered weeks later when the manual path was needed.

The Byzantine pattern here is authority laundering through documents. The runbook carried the author's name, so it carried the author's authority. The author's authority was never behind step eight, because the author never saw it. The document was honest about its contents and dishonest about its mandate, and no participant could tell the difference by reading it.

What makes this class dangerous is precisely that it does not look like an attack. There is no hostile text to detect, no suspicious instruction, no out-of-place imperative. The injected step is correct, well-explained, and consistent with everything around it. It fails only one test: a human with the right to decide never decided it.

---

## What actually caught it

Two habits, neither of them clever:

1. **The agent states what it is about to do before doing it.** The flip was not buried in a script log; it was announced in prose, in the status report, as a future action. Announced intent is the last place a human can intervene without reading code.
2. **The operator reads status reports as if they were pull requests.** One sentence out of twenty was wrong, and it was the one that mattered.

Nothing in the tooling flagged step eight. The code check passed. The runbook was from a trusted source. The only defense was a human noticing a word.

---

## Lessons

- **A document is data, not authorization.** Runbooks, plans, checklists and specifications, especially generated ones, describe what could be done. The mandate to do an irreversible thing comes from a human who owns the consequence, stated in the conversation, not inferred from a page.
- **Generated documents inherit the forwarder's authority and none of the forwarder's intent.** When someone shares a page their assistant wrote, assume they have not read every step. Ask about the irreversible ones explicitly.
- **Irreversible steps from documents get their own line.** In plans and status reports, list them separately as "from the document, not confirmed", so they stand out instead of hiding in a numbered list between two routine steps.
- **Announce intent in prose.** A status line saying "I will do X after Y" is cheap, and it is the only artifact a busy human will actually read before X happens.
- **Verification against code is necessary and insufficient.** Step eight was valid code-wise. Validity answers "can this be done", not "did anyone decide to do it".

**BAI lesson:** The most reliable prompt injection is the one nobody wrote on purpose. A generated runbook, forwarded unread, becomes an instruction with a human's name on it and no human behind it. Treat every irreversible step that arrived in a document as unconfirmed until the person who owns the consequence says so in their own words. **Documents describe; people authorize.**
