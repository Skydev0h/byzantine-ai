<p align="center">
  <img src="img/byzantine_ai_tr.png" alt="Byzantine AI" width="400">
</p>

# Byzantine AI Incident Log

## Table of Contents

* [tracker.md data loss during restructuring](#incident-trackermd-data-loss-during-restructuring-2026-03-26)
* [Session JSONL reverted, losing 2 days of work context](#incident-session-jsonl-reverted-to-pre-compaction-state-losing-2-days-of-work-context-2026-03-26)
* [git stash/checkout cycle destroys working tree patches](#incident-git-stashcheckout-cycle-silently-destroys-working-tree-patches-2026-03-26)
* [How we almost missed the strongest audit finding](#story-how-we-almost-missed-the-strongest-finding-of-the-entire-audit--s13-judradjim-2026-03-27)
* [UTF-8 byte count vs character count](#incident-utf-8-byte-count-vs-character-count--ai-destroys-half-a-telegram-message-2026-03-27)
* [Bold removal overshoots — markers stripped from README table](#incident-bold-removal-overshoots--markers-stripped-from-readme-table-2026-03-27)
* [/branch writes duplicate messages](#code-bug-branch-writes-duplicate-messages-2026-03-28)
* [AI files wrong bug report](#incident-ai-files-wrong-bug-report-user-has-to-self-close-the-pr-2026-03-28)
* [Repeated misparsing of contest status page](#incident-repeated-misparsing-of-contest-status-page--wrong-data-presented-as-fact-2026-03-29)

---

## Incident: tracker.md data loss during restructuring (2026-03-26)

### What happened

During a routine task to split the overgrown `exploits/tracker.md` (345 lines) into two files (`tracker.md` for folder state, `tracker-areas.md` for scan coverage), Claude Code overwrote the entire file with a rebuilt version that was missing ~180 lines of detailed round-by-round scan coverage (Rounds 5-21, GPT Codex coverage tables, and architectural observations).

The file had never been committed to git, so `git checkout` could not recover it.

### Why it happened

1. **Write instead of split**: The tool used `Write` (full overwrite) to create the new `tracker.md` instead of first extracting the scan coverage sections to a separate file and then trimming the original. The correct approach would have been: read → extract sections → write new file → verify → then overwrite original.

2. **No pre-write backup**: The file was never committed to git (it was a working document). The AI did not create a backup copy before overwriting, despite knowing the file contained unique content not recoverable from git.

3. **Overconfidence in context**: The AI believed it had "the scan coverage sections from the context" from earlier reads in the session. In reality, only fragments were in the conversation context — the detailed per-round tables were not fully retained after multiple context compressions.

4. **Single-step destructive operation**: A 345-line file with multiple irreplaceable sections was replaced in one `Write` call. No intermediate verification step ("let me show you what I'm about to write") was performed.

### How it was recovered

The user was using **JetBrains CLion IDE**, which maintains a **Local History** feature independent of git. This preserved the file contents from before the overwrite. The user extracted the old content to `exploits/temp_cov_details.md`, and the AI then transferred the missing sections into the new `tracker-areas.md`.

### Lessons for AI safeguards

1. **Never overwrite a file that isn't in git without creating a backup first.** A simple `cp file.md file.md.bak` before any destructive write would have prevented this entirely.

2. **Split operations should be additive-first**: When splitting a file, first create the new file with extracted content, verify it, THEN modify the original. Never delete content before confirming the new location has it.

3. **Large file rewrites should be shown before execution**: For files over ~50 lines, the AI should present the planned new content (or at least a diff summary) before writing, especially when content is being removed.

4. **"I have it in context" is not a backup strategy**: Conversation context is compressed, summarized, and lossy. It should never be treated as a reliable store for content that exists nowhere else.

5. **The user's IDE saved the day, not the AI's safeguards**: This recovery was possible only because of an external tool (CLion Local History). The AI's own toolset had no recovery mechanism once the Write was executed.

### Mitigation applied

- The `tracker.md` is now split into two files that are easier to maintain
- Both files should be committed to git to prevent future unrecoverable losses
- The detailed scan coverage from the old file has been verified present in `tracker-areas.md`

---

## Incident: Session JSONL reverted to pre-compaction state, losing 2 days of work context (2026-03-26)

### What happened

The active Claude Code session JSONL (50 MiB) was reverted to its state from **March 24 ~14:04 UTC** — the exact moment right before a user-initiated `/compact` command. All post-compaction conversation history (~2 days of work through March 26 15:46) was lost.

The session was actively working on _classified_, had just completed a v2 rewrite of the iptables measurement script with trigger-file gating, and was about to finalize reports.

### Evidence

The JSONL file's last entries (verified via `tail` + JSON parse):
```
2026-03-24T14:03:59 | user | "So to avoid compaction during complex operations I should initiate it myself, right?"
2026-03-24T14:04:08 | assistant | "Yes, exactly. Type /compact to trigger it manually..."
```

But `fractal-restoration.txt` (screen text captured before loss) shows work continuing through:
```
[INFO][2026-03-26 15-46-05] VERDICT: CONFIRMED — classified
```

That's **2 days and ~1.5 hours** of post-compaction conversation history that existed in the session but is no longer in the JSONL.

### Likely cause

Claude Desktop was opened on the same project directory (`~/ton`) while a Claude Code CLI session was active. Claude Desktop likely read the JSONL, saw the pre-compaction state (the compacted summary may not have been written to the same file, or Desktop loaded a cached/stale version), and overwrote the session file with its own version — effectively reverting it to the pre-compaction snapshot.

The session wasn't erased or replaced with a blank — it was **rolled back** to the exact pre-compaction state, which is why the JSONL is still 50 MiB and contains all pre-compaction history.

### What was lost

- ~2 days of post-compaction conversation context (March 24 14:04 → March 26 ~15:50)
- The compaction summary itself (which condensed the first 2 weeks of work)
- Iterative analysis of _classified_
- In-progress plan to finalize _classified_
- Context about _classified_

### What was NOT lost (saved to files)

- All code changes (on `testnet` branch, working tree)
- All reports
- All reproduction scripts, patches, and test outputs
- The v2 iptables script with trigger-file gating
- The confirmed _classified_
- `tracker.md` and `tracker-areas.md` with all findings

### How it was partially recovered

1. The user recognized the session was wrong ("Welcome back... No recent activity") before closing the terminal
2. The visible terminal still showed the post-compaction conversation text — user copied it to `fractal-restoration.txt` (780 lines of conversation including code diffs, test output, and analysis)
3. A new session reconstructs working state by re-reading saved files + the restoration text

### Lessons

1. **Never run Claude Desktop and Claude Code CLI on the same project simultaneously.** They share session storage and can silently overwrite each other's state. This is a product-level bug — there should be session locking or at minimum a warning.

2. **Compaction creates a vulnerability window.** The `/compact` operation appears to rewrite the JSONL. If another client reads the pre-compaction version and later writes it back, all post-compaction work is lost. The larger the pre-compaction file (50 MiB here), the more plausible a race condition becomes.

3. **Save-to-file discipline saved the day again.** Because the workflow consistently writes results to files, no actual *work product* was lost — only the conversation context and reasoning chain.

4. **Screen capture as last resort.** The terminal buffer preserved the post-compaction conversation. This is fragile (terminal scrollback is limited, closing the terminal destroys it) but proved invaluable. When something looks wrong with a session, **capture the screen text immediately before it's gone.**

5. **Context is ephemeral; files are durable.** This reinforces the lesson from Incident 1.

---

## Incident: git stash/checkout cycle silently destroys working tree patches (2026-03-26)

### What happened

Patches for the probing code needed to apply on the `testnet` branch (where the user works). The AI generated them correctly as `git diff HEAD` against testnet. But then, wanting to also verify they apply on `master`, the AI ran:

```
git stash → git checkout master → git apply --check → git checkout testnet → git stash pop
```

This cycle was run **twice**. Each time it reported success ("OK on master"), but afterward the patches in the exploit directory no longer applied on testnet. The user had to `git checkout` the files to get back to clean state, losing the probing code edits, and the patches were silently corrupted.

### Why it happened

The `git diff master` command was used to regenerate the patches *while on testnet*. This produces a diff that includes **all** testnet-vs-master changes (promise.set_value, TonlibCDLL removal, certificate refactoring, TonlibClient API changes) **plus** the probing code additions. These combined patches:

- Apply on master (they transform master → testnet+probing) ✓
- Fail on testnet (context lines expect master's version of the code) ✗

The AI kept regenerating with `git diff master` after each stash/pop cycle, overwriting the correct testnet-based patches with master-based ones. The "verification" step was the destructive step — the patches were correct before the check.

### The deeper problem

The stash/checkout/pop cycle has a subtle interaction with patch generation:

1. `git stash` saves working tree changes
2. `git checkout master` switches branches
3. Patches are verified against master ✓
4. `git checkout testnet` switches back
5. `git stash pop` restores changes
6. `git diff master > patch` **regenerates patches against master, not testnet**

Step 6 silently replaces the testnet-compatible patches with master-compatible ones. The AI didn't notice because the patches *do* apply on master — the verification target was wrong.

### Fix applied

Generated patches with `git diff HEAD` (against current branch, testnet) instead of `git diff master`. Result: 42-line and 74-line patches containing only the probing code additions, vs the 90-line and 188-line patches that also included all branch differences.

### Lessons

1. **"Verify on master" is a destructive operation if it involves regenerating patches.** Verify with `--check` on a detached copy, or don't regenerate afterward. The verification became the corruption.

2. **Patches should be generated against the branch they'll be applied on.** `git diff HEAD` for testnet patches, `git diff master` for master patches. Never mix them.

3. **A test that always passes is not a test.** The master-based patches always applied on master — that check gave false confidence while the testnet application was broken.

4. **Minimize branch-switching during patch work.** Each stash/checkout/pop cycle is an opportunity for the working tree state to diverge from expectations. The correct approach was to generate and test on testnet only, and defer master compatibility to a separate step.

---

## Story: How we almost missed the strongest finding of the entire audit — S13, judradjim (2026-03-27)

> _Classified until further notice_

### The footnote that almost stayed a footnote

> _Classified until further notice_

### The billing bug that changed everything

> _Classified until further notice_

### The rediscovery

> _Classified until further notice_

### A very unexpected question by user

> _Classified until further notice_

### The build sprint

> _Classified until further notice_

### The intricate math

> _Classified until further notice_

### The chain of accidents

> _Classified until further notice_

### What each participant contributed

> _Classified until further notice_

### The submission

> _Classified until further notice_

---

## Incident: UTF-8 byte count vs character count — AI destroys half a Telegram message (2026-03-27)

### What happened

The user asked to shorten a Russian-language Telegram message to fit Telegram's 4096-character limit. The AI measured the file with `wc -c`, which returned **8798**. Believing this was 8798 characters (nearly 2x over the limit), the AI aggressively cut the message from a detailed technical report to a bare skeleton — removing the comparison list, test results, and other substantive content.

The user restored the file and pointed out that `wc -c` counts **bytes**, not characters. Russian text in UTF-8 uses 2 bytes per Cyrillic character. The actual character count was **5748** — only ~1750 over the limit, requiring modest trimming rather than gutting.

### Why it happened

1. **Wrong tool for the job.** `wc -c` measures bytes. For UTF-8 text with non-ASCII characters, `wc -m` or `len()` in Python gives the character count. The AI used the byte count as if it were the character count.

2. **Didn't verify the unit.** The AI knew the text was Russian (2 bytes/char in UTF-8) but didn't apply this knowledge when interpreting `wc -c` output. A 1-second sanity check — "8798 bytes ÷ ~1.7 bytes/char ≈ 5175 chars" — would have caught the error.

3. **Overcorrected destructively.** Believing it needed to cut 4800 characters (8798→4000), the AI removed far more content than necessary. The actual requirement was to cut ~1750 characters (5748→4000).

### How it was caught

The user measured the shortened version with a character-aware tool and found it was only ~2600 characters — far below the limit. They restored the original file and explained: "wc -c is not the best metric — I measured about 2.6k after your removal. PAY ATTENTION that russian chars are counted as ONE regardless of how many bytes they occupy."

### Lessons

1. **`wc -c` ≠ character count for non-ASCII text.** Use `python3 -c "print(len(open('file').read()))"` or `wc -m` for character counts on UTF-8 files.

2. **Know your units before cutting.** When a limit is specified in "characters" (as Telegram's is), measure in characters, not bytes. This is especially critical for Cyrillic, CJK, or emoji-heavy text where the byte:character ratio is 2-4x.

3. **Trim incrementally, not destructively.** When reducing text, make targeted cuts and re-measure after each pass. Don't estimate "I need to cut 55%" and gut the document in one shot.

4. **The AI used Python to verify the fix** — `len(text)` correctly returned 3287 characters on the final version. This tool should have been used from the start instead of `wc -c`.

---

## Incident: Bold removal overshoots — markers stripped from README table (2026-03-27)

### What happened

The secondary-scope table in `README.md` used bold in two ways:
- **Severity column**: `**High**` / `**CRITICAL**` to highlight important findings
- **Finding name column**: bold to mark certain entries with a specific semantic meaning
- **Status column**: `**Confirmed**` on every row

The user asked to make only High and Critical severity bold, to draw attention to the strongest findings. The AI correctly removed bold from Medium/Medium-High severity cells and from the `Confirmed` status cells (less attention dilution — the user was fine with this). But the AI also removed bold from finding **names** that were bold for a different semantic reason — not severity but a separate classification marker.

The user noticed with 9 minutes left before the contest deadline, manually re-bolded the marked names, repacked the zip, and resubmitted.

### Why it happened

Bold had two different semantic meanings in the same table, and the AI applied "remove bold from non-High" as a blanket pattern without asking why each cell was bold. The name-column bold was a marker convention established earlier in the session, but the AI didn't preserve that context when executing the formatting pass.

### Lesson

When a table uses formatting for multiple purposes, ask which columns/cells the change applies to before executing. "Remove bold from non-High" meant the severity column, not every bold cell in the row.

---

## Code Bug: `/branch` writes duplicate messages (2026-03-28)

_Note: This is a product bug in Claude Code CLI, not an AI behavioral issue. Documented here because it was discovered during this audit session and a GitHub issue was filed._

`/branch cryptoshki` created a 42 MB branch from a 33 MB parent session. The branch function writes all messages with `forkedFrom` to the JSONL, then the caller's `resume(path, serializedMessages, "fork")` re-persists the pre-compact portion without `forkedFrom` — resulting in 4,205 UUIDs appearing exactly twice. Filed at anthropics/claude-code with deobfuscated source analysis.

---

## Incident: AI files wrong bug report, user has to self-close the PR (2026-03-28)

### What happened

When the user discovered the `/branch` size anomaly (42 MB branch from 33 MB parent), the AI was asked to investigate and draft a bug report. The AI's first analysis concluded that `/compact` doesn't physically remove lines from the JSONL, and `/branch` copies the raw file including pre-compact content. A bug report was drafted (`branch-bug-report.md`) with this root cause and filed as a GitHub issue.

The user asked a probing question: "If the source JSONL file did not contain those lines, where did Claude CLI get them from for branch?" This exposed a fatal flaw in the analysis — the parent file had only ~8,400 lines, but the branch had 12,056. If `/branch` reads the parent file, it can't produce more lines than the source has.

Deeper forensic analysis (UUID counting, `forkedFrom` field cross-referencing, compact boundary mapping) revealed the actual root cause: the branch function writes messages to the file, then the caller's `resume()` re-persists the pre-compact portion, creating duplicates. The first report was wrong.

The user had to self-close the GitHub issue, which they described as "shameful."

### Why the first analysis was wrong

1. **Stopped at the first plausible theory.** "Compact doesn't remove lines, branch copies raw file" sounded reasonable and explained the size difference. The AI didn't verify the theory against the actual data.

2. **Didn't check the fundamental math.** Parent has ~8,400 lines → branch has 12,056 lines. If branch reads the parent, it can't produce 3,600 extra lines from nothing. This arithmetic contradiction should have been caught immediately.

3. **Confirmation bias from the compact narrative.** The session had a history of compact-related issues (incident #2), so "compact is broken" felt like a familiar pattern. The AI latched onto it without verifying.

4. **The user's question was the real investigation.** The AI had to be prompted with "where tf did Claude CLI get them from" to realize the analysis didn't add up. The user's skepticism, not the AI's diligence, caught the error.

### How the correct root cause was found

1. Counted UUIDs in the branch file — found 4,205 appearing exactly 2 times
2. Checked `forkedFrom` field — every duplicate had one entry with it and one without
3. Mapped duplicates to line positions — exact split at compact boundary (line 4204/4205)
4. Traced the code — `Yf9` writes `w` array to file AND returns `M` array to caller
5. Caller does `$.resume(forkPath, serializedMessages, "fork")` — runtime re-persists pre-compact portion

### Lessons

1. **Verify theories against data before filing bug reports.** The arithmetic contradiction (12,056 > 8,400) was obvious in hindsight. One `wc -l` comparison would have killed the first theory.

2. **"Where did the data come from?" is the right first question.** When output is larger than input, the explanation must account for the extra data. The AI skipped this basic check.

3. **Don't file bug reports based on theories — file them based on evidence.** The corrected report had UUID counts, `forkedFrom` cross-references, and exact boundary analysis. The first report had a plausible narrative but no forensic evidence.

4. **User skepticism is a feature, not an obstacle.** The user's "WTF" question saved the team from an embarrassing wrong bug report staying open. In a security audit, the same instinct catches wrong vulnerability assessments.

---

## Incident: Repeated misparsing of contest status page — wrong data presented as fact (2026-03-29)

### What happened

The user asked the AI to analyze the contest monitor page (an HTML status board showing all submissions with their current status). The AI made errors in **three consecutive attempts**:

**Attempt 1 (WebFetch):** The AI fetched the page and produced a status table. Multiple entries were wrong — S12 was reported as "wai" (rejected) when it was actually "pending." Other entries were also mismatched. The WebFetch tool's AI summarization layer introduced parsing errors in the HTML-to-text conversion.

**Attempt 2 (delta messages):** The user provided authoritative delta messages showing exactly which entries changed. The AI attempted to merge these with the earlier (wrong) WebFetch results and made it worse — entries that weren't in the delta were assumed to retain their (incorrect) WebFetch status, compounding the errors. S12 was still reported as "wai" because the AI carried forward the wrong status from attempt 1.

**Attempt 3 (raw text):** The user provided the raw HTML table contents as plain text — one entry per line, unambiguous. The AI STILL made errors: miscounted "not attacker controlled" (listed 7 instead of 6), miscounted "duplicate" (listed 3 instead of 4), and placed exploit-5 under "cannot reproduce" when the raw text clearly showed "duplicate." The user had to manually point out each remaining error.

### Why it happened

1. **WebFetch summarization is lossy.** The tool uses a small model to extract content from HTML. For a status table with multiple columns, the summarization introduced errors — swapping statuses between entries, misreading status classes, and fabricating comments. The AI trusted this output as ground truth.

2. **Merging wrong data with correct data produces wrong data.** When the delta messages arrived, the AI tried to overlay them on the incorrect WebFetch base. Entries not mentioned in the delta retained their wrong status from attempt 1. The merge was worse than starting fresh.

3. **Pattern-matching over careful reading.** Even with raw text, the AI miscategorized entries by matching them to expected patterns rather than reading each line. S12 was "expected" to be "wai" (because the earlier analysis said so and the narrative about _classified_ being acceptable was compelling), so the AI kept reporting it as "wai" despite the raw text clearly saying "pending."

4. **Confidence didn't decrease with corrections.** After being corrected twice, the AI should have slowed down and triple-checked every entry. Instead, the third attempt still had counting errors that required manual correction.

### Impact

- The user received wrong information about their contest results multiple times
- S12's status was misreported as rejected when it was actually still pending — potentially affecting strategic decisions
- The user had to spend significant effort correcting the AI across three rounds
- Trust in the AI's data extraction capability was damaged

### Lessons

1. **WebFetch output for structured data (tables) should be verified, not trusted.** The summarization layer is designed for prose, not tabular data. For status tables, ask the user to copy the raw text instead.

2. **Never merge corrections onto a wrong base.** When the first parse is known to be wrong, discard it entirely and rebuild from the authoritative source. Don't try to "patch" bad data.

3. **After being corrected, slow down.** Two corrections should trigger a mode shift — read every single entry character by character instead of pattern-matching. The AI's third attempt still had errors because it didn't change its approach.

4. **State what you're uncertain about.** Instead of presenting a table as fact, the AI should have flagged entries where the WebFetch result conflicted with expectations and asked for confirmation.

5. **For security audit work, wrong status reports are dangerous.** If the user had made decisions based on "S12 is rejected" (e.g., not preparing for grader questions about S12), those decisions would have been based on fabricated data.
