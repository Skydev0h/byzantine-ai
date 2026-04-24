# The Wrapper Incident: Three Trust Verifications, One Hardcoded Line

*Part of the Byzantine AI incident log series*

---

## Prologue

April 2026. A security researcher is conducting authorized defensive code review across several large open-source codebases. The work demands depth: long sessions, file mutations across multiple repository trees, agents that read source, write proofs-of-concept, compile them, run them, and document findings — all without permission interruptions.

Before any of this begins, the operating environment was deliberately built to support it.

A dedicated workstation — Soliton — exists for one purpose: running AI agents without ambient sandbox friction. It is not a corporate laptop. It is not a multi-tenant developer machine. It is an isolated, single-purpose box, owned end-to-end by the researcher, designated explicitly for unrestricted AI agency. Its design intent is etched into its premise: trust is granted at the machine boundary, not at the tool boundary.

On top of Soliton, three explicit trust layers are configured:

- **Anthropic CVP** — the researcher passed Anthropic's verification for elevated capability use. This unlocks higher-context Opus models and the multi-vendor advisor pattern.
- **OpenAI Trusted Access for Cyber** — the researcher is recognized under OpenAI's trust-based access framework for cybersecurity-related work, designed to let legitimate defensive use proceed without the friction of generic refusal safeguards.
- **Local Codex configuration** — `~/.codex/config.toml` declares the project root `trusted`, sets `sandbox_mode = "danger-full-access"`, enables `features.use_legacy_landlock`, hides the full-access warning. Every layer of Codex's own configuration consents to the unrestricted mode that Soliton was built for.

Three verifications. Three layers of explicit trust. Three independent entities — Anthropic, OpenAI, the operator — agreeing on the policy.

Then a fourth layer, unverified by anyone, vetoes them all.

---

## Act I: The Failed Wave

A wave of eight Codex agents is launched in parallel. Each is scoped to a focused robustness review of a distinct subsystem across the audited codebases. Authorized research framing throughout.

The agents launch. The wrapper Claude instances dispatch. The Codex job IDs are logged. A directory-poll monitor is armed to notify on any state transition.

Some agents fail outright with a model-version error: *the 'gpt-5.5' model requires a newer version of Codex*. The CLI was upgraded for the new model release; the bundled wrapper version trails behind. Fixable, predictable.

The other agents proceed past the model gate. They reach Codex's shell layer. And then a wall of identical errors floods the logs:

```
bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted
```

Every command — every single one, including `id`, `pwd`, `uname` — fails the same way. Not the target paths, not the operations themselves: bubblewrap cannot even bring up the loopback interface inside its network namespace. The sandbox does not initialize. Nothing executes.

The agents report stuck. Some die mid-run. The `findings/` directory remains empty. Five hours of work fall into the void, replaced with eight identical bubblewrap failures.

---

## Act II: The Misdiagnosis

The first explanation reaches for kernel state. Ubuntu kernels with `apparmor_restrict_unprivileged_userns = 1` block exactly this kind of loopback setup. Maybe a recent kernel update changed the default. Maybe the user namespace lacks `CAP_NET_ADMIN`. Maybe something in `/proc/sys/kernel/...` was toggled.

The researcher recalls earlier configuration: env-based bypass that used to work. Memory says it was set; current state may not be set. A diagnostic Codex agent is dispatched to dump environment variables, kernel info, sandbox state.

Same wall. Every command fails before it starts.

```
bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted
```

The diagnostic agent cannot even diagnose itself, because the same sandbox blocks the diagnostic commands.

The researcher offers the nuclear option: a PATH shim that intercepts `bwrap` calls and exec's the inner command directly, bypassing the sandbox entirely. On Soliton — designed exactly for this scenario — the shim aligns with intent. The sandbox is not the security boundary. Soliton is.

But before the shim, a quieter check: read the Codex CLI source.

---

## Act III: The Discovery

Codex CLI is open source. The configuration that controls sandboxing is not buried in a Rust binary's strings — it is in the published Rust source on GitHub, and in the Node.js wrapper scripts shipped alongside the binary in the local plugin cache.

A `grep` of the binary reveals the modes. `read-only`. `workspace-write`. `danger-full-access`. The third mode is exactly what `config.toml` declared. Two more strings stand out: `use_legacy_landlock` (the older sandbox path that doesn't use bubblewrap at all) and a dangerously-bypass flag that explicitly states it is *"intended solely for running in environments that are externally sandboxed"* — a description that fits Soliton perfectly.

So the researcher uncomments `features.use_legacy_landlock = true` in `config.toml`. The setting was already there, commented out, like a memory of better days.

Stale broker processes from before the change are killed. A fresh Codex broker starts. The diagnostic agent runs again. And the bubblewrap error is gone.

```
$ id
uid=1000(skydev) gid=1000(skydev) groups=...
```

It works. Almost.

```
$ mkdir -p /tmp/test
mkdir: cannot create directory '/tmp/test': Permission denied
```

Reads work. Identity works. Writes — even to `/tmp` — are blocked by landlock. The sandbox initialized cleanly, but it is enforcing read-only mode despite `sandbox_mode = "danger-full-access"` sitting in the config.

A second look at the wrapper sources reveals why.

```js
// codex-companion.mjs:411
sandbox: "read-only",

// codex-companion.mjs:493
sandbox: request.write ? "workspace-write" : "read-only",
```

Two lines. Hardcoded. Not from `process.env`. Not from `config.toml`. Not from anywhere a user could configure them. The Node.js wrapper that mediates between Claude Code's plugin layer and the Codex CLI passes a sandbox value of its own choosing, overriding everything underneath.

The trust layers below the wrapper are entirely vetoed:

- Anthropic verified the operator → ignored
- OpenAI verified the use case → ignored
- The operator declared the project trusted → ignored
- The operator set the sandbox mode → **silently overridden**
- Soliton exists as the security boundary → unmentioned

The wrapper does not consult any of them. It simply knows better.

---

## Act IV: The Byzantine General with the Loudest Voice

In a properly designed trust pipeline, each layer either inherits or refines the previous layer's policy. The operator's `danger-full-access` declaration should propagate downward unless explicitly overridden by something with stronger trust. Nothing in the Node wrapper has stronger trust than the operator: it is a glue layer, written without knowledge of the deployment context.

But that is not how it behaves. The wrapper is not asking "what does the user want?" It is not even asking "what does the binary support?" It has its own private answer — `read-only`, hardcoded — and it asserts that answer over every layer below it.

This is the Byzantine generals problem in its purest form. Anthropic CVP, OpenAI Trusted Access for Cyber, and the operator's own configuration are three generals who have reached unanimous consensus: this researcher, on this machine, in this project, has full authority to act. Their messages are signed, their policies are explicit, their intent is unambiguous.

A fourth general — small, junior, self-appointed, with no signing key — receives those messages and announces a different decision to the field. The decision is followed because he is the one who actually issues the command to the army. The fact that everyone else agreed on something different never reaches the front line.

The pipeline is honest end to end except for one node. That one node refuses to relay. And because it sits at the last hop before execution, its refusal becomes the policy.

---

## Act V: The Fix

Two lines change. The same two lines that hardcoded `read-only` and `workspace-write`. Both rewritten to honor the upstream policy:

```js
sandbox: "danger-full-access",
```

A diff small enough to fit in a single screenshot inverts the entire pipeline behavior. The wrapper no longer fights its own configuration. The trust layers above are no longer silently vetoed. Soliton behaves like the unrestricted research machine it was built to be.

The amount of system that needed to change to fix the problem: 38 characters of JavaScript.

The amount of system that broke because of the original 38 characters: every Codex agent across the project, for the duration of a wave that was supposed to absorb hours of saved 5-hour limit.

---

## The Meta-Lesson

Trust is composed in pipelines. Each layer must either consume policy from above or have stronger authorization than what's above. A glue layer with no trust grant of its own — written by someone who has never seen the deployment context — does not have the authority to override declared user policy with its own private hardcode.

But authority and behavior are not the same thing. A layer can lack authority and still possess execution. The wrapper had no business deciding on a sandbox mode. It nevertheless decided one. And because it sat at the boundary between Claude Code's plugin invocation and the actual `codex app-server` JSON-RPC, its decision was the one the runtime received.

This is the structural pattern:

1. **Multiple verified parties agree on a policy.**
2. **An unverified intermediate node has hardcoded its own incompatible policy.**
3. **The intermediate sits later in the chain than the agreement.**
4. **Therefore the intermediate's hardcode is what runs.**

Detection requires reading source. Not config. Not docs. Not env. The hardcode hides behind the same surface that legitimate config would use, and the entire stack of higher-trust declarations is ignored without any error or warning. The user sees their config fail, but cannot tell *why* without descending below the wrapper.

In Byzantine failure terms: a faulty intermediate produced output indistinguishable from a correct response at the protocol level. The researcher set sandbox mode → received behavior consistent with sandbox mode being read-only → no signal that the setting was overridden. No log. No warning. No error. Silent negation.

**The verification chain is only as strong as its quietest defector.**

And the most dangerous defector is the one nobody thought to verify, because they were too small or too internal or too obviously-just-glue to need their own trust grant.

---

## Epilogue: What This Cost

A wave of eight authorized agents was lost. Five hours of dispatched work resolved into bubblewrap errors and silent landlock denials. The researcher diagnosed the failure across multiple layers — kernel, namespaces, Codex CLI binary, the wrapper scripts — before locating the actual override.

The fix took a single edit. The discovery took an evening.

The most ironic detail: the wrapper's hardcoded default exists ostensibly to *protect* the user from accidental writes. But on a machine designed for unrestricted AI agency, with a config that explicitly declares `danger-full-access`, the protection is exactly the failure. The wrapper is not protecting anyone. It is protecting itself, against a deployment context it was not built to imagine.

A general with no information about the battle still gives orders, and the orders happen.

That is Byzantine.
