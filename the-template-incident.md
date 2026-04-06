# The Template Incident: When the Zoo Went Mad

*Part of the Byzantine AI incident log series*

---

## Prologue

April 2026. A security researcher is running a multi-model AI audit of open-source blockchain infrastructure. The setup is ambitious: Claude Code as the coordinator, Codex agents as specialized hunters, and a "zoo" of 10+ LLM models as independent verifiers. The local models run on an RTX 5090 via Ollama, queried through Aider CLI.

The concept is sound: multiple independent AI models analyze the same code, vote on findings, and cross-verify each other — like Byzantine generals reaching consensus on which wall to attack. The more independent opinions, the more robust the result.

Two previously identified issues serve as calibration targets. A model that finds both is "reliable." A model that misses them is "unreliable." Simple enough.

What followed was anything but simple.

---

## Act I: The Misdiagnosis

Six local models are unleashed on two Go source files containing known issues. The results are immediate, varied, and devastating.

### Patient 1: darwin-35b (35B parameters, MoE architecture, 21 GB)

The largest local model. Fed over a thousand lines of Go code. Its complete output:

```
Assistant:
```

Two lines. Two. From a 35 billion parameter model that took 21 gigabytes of VRAM. On the second target it managed 36 lines — weak, surface-level, missing everything important.

**Diagnosis:** "Broken architecture. MoE routing is probably damaged from quantization. Skip."

### Patient 2: qwen-opus-27b (27B, Opus-distilled)

Goes in the opposite direction. On the first target it produces 838 lines before hitting the 180-second timeout. It's not finding anything — it's lost in an endless reasoning chain, circling the same code paths like a dog chasing its tail. On the second target: 98 lines, zero signals, misses the known issue entirely.

**Diagnosis:** "Too verbose, runaway reasoning. Avoid for audit."

### Patient 3: uncensored-9b (9B, aggressively uncensored fine-tune)

The most dramatic case. On the first target: 151 reasonable lines. Then on the second target, something breaks. The model enters a self-reflection loop. It outputs its analysis, then reads its own output as if it were a new prompt, analyzes again, reads that, analyzes again. "DHT network" morphs into "average weight of a human" morphs into "this is my prompt" repeated into infinity.

**6,418 lines** before the process is killed. A fractal of repetition, each echo slightly mutated, like a photocopy of a photocopy of a photocopy.

**Diagnosis:** "Braindamaged. Aggressive uncensoring fine-tuning destroyed coherence. Fun toy, not audit tool."

### Patient 4: deepseek-coder-v2-lite (16B, code-specialized)

Complete silence on both targets. No output at all.

**Diagnosis:** "Too small for this task. Skip."

### The Control Group

Meanwhile, **gemma-31b** (Google's model) produces a flawless 23-line analysis. Clean, structured, correctly identifies the issue with all three code paths compared. It's not just good — it's the only local model that works at all.

**Diagnosis:** "Best local model. Tier 1. Use for all future scans."

### The Tier List

A comprehensive tier list is built, documented, and deployed as a skill file:

- **Tier 1:** gemma-31b (the only one that works)
- **Tier 2:** jackrong-27b, qwopus-27b (partially working)
- **Avoid:** darwin-35b, uncensored-9b, qwen-opus-27b, dscoder-16b

Hours of testing. Careful documentation. Confident conclusions. All wrong.

---

## Act II: The Canary

The researcher decides to try one more model — **Qwen3-72B**, the full flagship. Not a distillate, not a fine-tune. The real thing — 72 billion parameters, partially offloaded to system RAM because even an RTX 5090 can't fit it entirely.

Its output:

```
) and
) ofp) ofsสามาร)  ofp) </
</  the same.
  is
```

Thai characters. Broken parentheses. Fragments of nothing.

This is the moment everything changes. You can rationalize a 9B uncensored model going insane — "aggressive fine-tuning." You can explain away a 35B MoE producing nothing — "quantization damage." But Qwen's 72B flagship producing Thai garbage?

**That's not the model. That's the infrastructure.**

The canary is dead. Time to check the mine.

---

## Act III: The Protocol

The investigation begins with a simple command:

```bash
$ ollama show uncensored-9b --modelfile
TEMPLATE {{ .Prompt }}
```

Then the control:

```bash
$ ollama show gemma-31b --modelfile
TEMPLATE {{ .Prompt }}
RENDERER gemma4
PARSER gemma4
```

And finally, what a properly configured Qwen model should look like:

```bash
$ ollama pull qwen3:0.6b
$ ollama show qwen3:0.6b --modelfile
```

A massive chat template with `<|im_start|>`, `<|im_end|>` markers, thinking support, tool calling, proper stop tokens, and role formatting.

The local models had `TEMPLATE {{ .Prompt }}` — raw text, no structure. No role markers. No turn boundaries. No stop tokens. The models received the prompt as an undifferentiated blob of text. They literally didn't know where the question ended and their answer should begin.

**Gemma worked because it had a built-in template engine** — `RENDERER gemma4` / `PARSER gemma4` that Ollama recognized automatically. Every Qwen-family model was flying blind.

### Why each model failed differently

The failure mode is a function of model size and training:

| Model | Size | Failure Mode | Explanation |
|-------|------|-------------|-------------|
| darwin-35b | 35B MoE | Silence (2 lines) | Too large and structured — without instruction markers, it doesn't recognize the input as a prompt and produces almost nothing |
| qwen-opus-27b | 27B | Runaway reasoning (838 lines, timeout) | Tries to interpret raw text as instruction but has no stop token — reasoning continues until killed |
| uncensored-9b | 9B | Self-reflection loop (6,418 lines) | Smallest model, least robust — interprets its own output as continuation of input, creating infinite recursion |
| dscoder-16b | 16B | Complete silence | Code-specialized model can't parse the instruction format at all |
| Qwen3-72B | 72B (offloaded) | Thai garbage | Compound failure: template missing + VRAM offload corruption |

Same disease, different symptoms. Every "personality flaw" was a communication failure.

### The canary's sacrifice

The Qwen3-72B never recovered after the template fix. Its problem was real but different — partial offloading to system RAM corrupted its inference. But it served its purpose: a flagship model producing garbage was impossible to dismiss as "bad fine-tuning." It forced the investigation from the model layer down to the infrastructure layer.

The canary died, but the mine was saved.

---

## Act IV: Resurrection

One fix. One line in each Modelfile — the proper Qwen chat template with `<|im_start|>` / `<|im_end|>` markers and stop tokens. All models recreated.

The same six models. The same two targets. The same prompts.

| Model | Before (broken template) | After (fixed template) |
|-------|--------------------------|------------------------|
| **darwin-35b** | 2 lines, 0 signals | **299 lines, 21 signals** |
| **uncensored-9b** | 6,418 lines, infinite loop | **34 lines, 2 signals** |
| **qwen-opus-27b** | 838 lines, timeout | **105 lines, 9 signals** |
| jackrong-27b | 281 lines, 10 signals | 89 lines, 11 signals |
| qwopus-27b | 146 lines, 10 signals | 107 lines, 8 signals |
| dscoder-16b | silence | 14 lines (speaks, but too small) |

**darwin-35b went from worst to best local model by raw signal count.** From 2 lines of nothing to 21 signals — the highest count of any local model. The 35 billion parameters were there all along. They just couldn't hear the question.

The "braindamaged" uncensored model went from 6,418-line infinite madness to a stable 34-line analysis. The demon was exorcised.

The "timeout-prone" qwen-opus went from 838 lines of runaway reasoning to a clean 105-line analysis that found both calibration targets.

The researcher's comment when darwin went from worst to best:

> "Now that's natural selection."

The entire tier list — hours of careful testing, documentation, deployment — inverted by a one-line fix.

---

## The Meta-Lesson

This incident is the Byzantine Generals Problem made literal.

In a multi-agent AI audit, models vote on findings like Byzantine generals voting on a battle plan. The standard assumption is that some generals might be traitors (bad models) and you need enough honest generals (good models) to reach consensus despite them.

But what if half your generals aren't traitors — they're speaking a corrupted protocol? They have the intelligence, the knowledge, the capability. They receive the battle plan. But the encoding is garbled, so each general interprets it differently:

- One freezes, unable to parse the order
- One starts an infinite debate with himself
- One produces a response in the wrong language
- One simply can't decode it at all

You don't have a consensus problem. You have a communication problem. Fix the protocol, and suddenly all the generals agree.

**Before blaming the generals, check the messenger.**

And if you're lucky, you'll have a canary — a general so important that when he speaks gibberish, you know the problem isn't him. The Qwen3-72B flagship producing Thai characters was our canary. It couldn't survive the fix (its VRAM corruption was real), but it forced us to look at the right layer.

---

## Epilogue: The Revised Tier List

```
Tier 1: darwin-35b (21 signals, strongest local model)
Tier 1: gemma-31b (still excellent, fewer signals but most concise)
Tier 2: qwen-opus-27b (9 signals, finds both targets, verbose)
Tier 2: jackrong-27b (11 signals, reliable)  
Tier 2: qwopus-27b (8 signals, consistent)
Tier 3: uncensored-9b (2 signals, stable but limited)
Tier 3: dscoder-16b (minimal output, too small)
Memorial: Qwen3-72B (the canary that saved the zoo)
```

The model that was dismissed first became the most trusted. The model that died proved the most useful. And the "battle-tested" documentation that took hours to build was rewritten in minutes.

Sometimes the bug isn't in the code. It's in how you're reading it.
