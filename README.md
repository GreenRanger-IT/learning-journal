# Learning Journal — Building Production SRE Tooling

> Architecture decisions, Kubernetes concepts, and engineering thinking accumulated
> while building AI-assisted SRE tooling from scratch. Each entry is a future blog
> post, conference talk, or YouTube video.

---

## Architecture & Design Thinking

### The dual-layer pattern: deterministic CLI + AI reasoning

**What I learnt:** The right architecture for AI-assisted tools is two separate layers
with a clean contract between them — not one big AI agent that does everything.

**The contract:** CLI tool emits structured JSON. AI layer consumes JSON, adds reasoning.

**Why this matters:** The CLI runs in CI at 3am when nobody is awake. It must be
deterministic and fast. The AI layer runs interactively — it can be slower, richer,
and conversational. Mixing them forces you to choose between CI-safe and intelligent.
Separating them gives you both.

**Applied to:** Any tool that needs to work offline/in-CI AND provide rich reasoning.
This pattern applies beyond SRE — data pipelines, code review tools, testing frameworks.

---

### Severity as an exit code

**What I learnt:** A health checker without a machine-readable result is just pretty
output. Exit codes are the API of shell tools.

**The pattern:** `0`=all clear, `1`=warnings (proceed with caution), `2`=critical (block).

**Mental model:** Think of exit codes as HTTP status codes for shell tools.
`0`=200 OK, `1`=warning (like a 4xx you can retry), `2`=critical (like a 5xx that blocks).

**Where I use this:** `kubectl-sentinel` exits 0/1/2. CI pipelines check the exit code,
not the output. This is how `grep` works, how `test` works, how every good Unix tool works.

---

### Classify before you recommend

**What I learnt:** "Pod is OOMKilled → raise the memory limit" is wrong advice ~40% of
the time. The right P1 action depends on WHY it was killed, not just WHAT happened.

**The classification pattern:**
- `LIMIT TOO LOW` → raise the limit (safe to action)
- `POSSIBLE LEAK` → do NOT raise the limit without profiling (raises limit → leak fills the bigger limit → still crashes)
- `INPUT TRIGGERED` → fix the input handling (raising limit is a sticking plaster)

**Generalised:** This applies to every finding type — CrashLoopBackOff (exit code
matters), ImagePullBackOff (auth vs network vs quota), pending pods (resource vs
affinity vs PVC). Always ask "what kind of problem is this?" before "what should I
do about it?"

**Design thinking:** "Jump to solution" is a bias. Diagnosis before prescription.
This is what separates a good incident responder from a great one.

---

### Causation chains vs correlation

**What I learnt:** Finding A and Finding B are correlated doesn't mean A caused B.
When your triage tool says "OOMKill → service degradation", it needs to explain the
mechanism.

**The Via: pattern:** Every causal link in a chain must state HOW the failure
propagated — synchronous HTTP, async queue, shared node resource, shared database.
If you can't state the mechanism, the link is inferred, not confirmed.

**Practical rule:** Mark inferred links with `⚑ INFERRED` and give the engineer one
command to verify it. Don't assert causation as fact without log or trace evidence.

---

### Building for the 3am reader

**What I learnt:** Every design decision in an SRE tool should be made as if the
person reading the output has been awake for 3 hours, is stressed, and needs to act
in 5 minutes.

**This changes:**
- Report order: most critical findings first, not alphabetical or logical
- Language: `"Secret app-secrets missing — entire stack down (11 deployments at 0 replicas)"` not `"FailedMount events detected"`
- Commands: give the exact kubectl command, not "check the pod logs"
- Classification: classify before recommending so the stressed engineer doesn't make a wrong call

---

## Kubernetes Technical Concepts

### Multi-container pods and container-level OOM

**What I learnt:** A pod can have multiple containers with independent memory limits.
When a pod gets OOMKilled, you need to identify which CONTAINER was killed — not just
which pod.

**How to find it:** `kubectl describe pod <name> -n <ns>` → look at "Last State" for
each container. The container with `OOMKilled` in Last State and a runtime of <5min
before kill is the victim.

**Live finding:** A two-container pod had one container with a 210Mi limit crashing
at ~2min and a second container with a 2Gi limit that was healthy. The pod name is
not the container name. You must check each container's Last State individually.

---

### HPA min==max means no autoscaling

**What I learnt:** A HorizontalPodAutoscaler with `minReplicas == maxReplicas` is
functionally inert. It's a no-op. The HPA cannot scale up because max is already
the current count; cannot scale down because min is already the current count.

**Why this matters in incidents:** If an HPA is at "ceiling" and
`minReplicas == maxReplicas`, the fix is to raise `maxReplicas`, not to investigate
HPA health. The HPA is working exactly as configured — the configuration is wrong.

```bash
kubectl get hpa -n <namespace>
# If MINPODS == MAXPODS, autoscaling is disabled by configuration
```

---

### SLO burn rate math

**What I learnt:** A 2x burn rate on a 1-hour window means your error budget is being
consumed twice as fast as it should be. An 8x burn rate on a 24-hour window is a slow
burn — it won't exhaust your monthly budget in 24 hours, but it will in ~3 days if it
continues.

**Why the window matters:**
- Short-window burn rate (1h) catches fast burns — service is down right now
- Long-window burn rate (24h/72h) catches slow burns — service is degraded but not down, accumulating errors slowly

**Example:** An 8x burn rate on a 24-hour window → not an emergency, but exhausts
the monthly error budget in ~3.7 days if unaddressed. P3 severity is correct.

**What I'd tell a junior SRE:** When you see a burn rate alert, don't look at the
current error rate — look at the WINDOW error budget consumption. The alert tells you
the rate of consumption, not the current state.

---

### GCP Cloud Monitoring SLO alerts

**What I learnt:** GCP Cloud Monitoring SLO alerts use `select_slo_burn_rate` as the
metric. They fire against the window burn rate, not the raw error rate. The alert
resolves itself when the burn rate drops below the threshold — no manual action needed
to close it.

**Reading the alert title:** Environment tokens in the alert title (e.g., `[DEV]`,
`[PROD]`) can be mapped to the affected namespace and cluster. Build a lookup table
to automate this mapping in your triage tooling.

---

## Bash & Shell Scripting

### Bash 3 compatibility (macOS)

**What I learnt:** macOS ships bash 3.2 (from 2007) because bash 4+ is GPLv3 and
Apple doesn't ship GPLv3 software in the OS. This means:
- No `declare -A` (associative arrays)
- No `mapfile` / `readarray`
- No `${var^^}` for uppercase

**The fix:** Use parallel indexed arrays (`NAMES[]`, `VALUES[]`) instead of
associative arrays. Index by position, not by key.

**Where this bit me:** A section summary table that worked on Linux failed silently
on macOS. Rewrote from `declare -A` to parallel arrays for true portability.

---

### fd-redirect buffering pattern

**What I learnt:** When you need to collect output in the background (to reorder it)
while keeping global variables accessible, the Unix answer is fd redirection.

**The pattern:**
```bash
exec 3>&1          # save stdout to fd 3
exec 1>"$tmpfile"  # redirect stdout to file
# run your checks — output goes to tmpfile, globals still update
exec 1>&3          # restore stdout
cat "$tmpfile"     # replay the buffered output
```

**Where I use this:** `kubectl-sentinel` prints a status banner with CRITICAL counts
BEFORE the section details. But we don't know the CRITICAL count until all checks
have run. Solution: buffer section output to a temp file, print the banner first,
then replay the file.

---

## Python Engineering

### stdlib-only Python for distribution

**What I learnt:** A Python tool with zero pip dependencies is dramatically easier to
distribute than one with a `requirements.txt`. Users don't need virtualenvs. CI
doesn't need pip install steps. The tool works on any system with Python 3.x.

**Constraint:** no `requests` (use `urllib`), no `rich` (use ANSI codes directly),
no `PyYAML` (use `json`), no `click` (use `argparse`).

**Result:** `pip install` is never in the README. `curl | bash` is the install path.

---

### Exit code contracts in CLI tools

**What I learnt:** The exit code is part of the CLI tool's public API. If you change
it, you break every script that calls your tool. Document exit codes in the module
docstring, not just in `--help`.

```python
"""
Exit codes:
  0 — triage complete, valid output
  1 — namespace unknown; user must answer
  2 — snapshot missing or invalid
  3 — no findings match the alert (low confidence)
"""
```

---

## AI & LLM Engineering

### Causation reasoning is not something you can hard-code

**What I learnt:** A CLI tool can identify "pod is OOMKilled" deterministically. It
cannot say "this OOMKill caused the service degradation because the scheduler pod is
in the same request path as the API". That requires understanding the system
architecture — which changes, and which can't be expressed in a lookup table.

**Why this separates LLM reasoning from rule-based reasoning:** LLMs can reason about
architecture from context (alert descriptions, service names, namespace topology). A
lookup table of "OOMKill → cause service degradation" would apply incorrectly in 90%
of cases.

**Practical lesson:** Use rules (CLI) for what is always true. Use LLMs for what
requires context.

---

### Confidence as a first-class output

**What I learnt:** A triage report that doesn't state its confidence is dangerous.
The engineer either over-trusts it (acts on the wrong root cause) or under-trusts it
(ignores it). Making confidence explicit forces the tool to reason about what it
doesn't know.

**The format:** `Confidence: HIGH | MEDIUM | LOW — <one-sentence reason>`

| Level | Meaning |
|-------|---------|
| HIGH | Direct match between alert symptom and CRITICAL finding in the expected namespace |
| MEDIUM | Indirect evidence, or low-confidence correlation |
| LOW | Alert symptom does not match any finding — cluster state collected for context only |

---

### PD intelligence vs PD metadata

**What I learnt:** The purpose of PagerDuty historical data in triage is NOT to get
alert metadata. It's to answer: "Is this a one-off or have we seen this before? Did
a previous fix hold? Are related services degrading simultaneously?" That context sets
the confidence of the causation chain.

**Why this matters:** "4th time in 6 months — last: 2026-02-14" tells you the P3
action is not "monitor and close". It tells you the root cause was never fixed or
keeps recurring. That changes the escalation path.

---

## Career & Professional Development

### The 5-year SRE to platform engineer transition

**What I learnt:**
- SRE solves incidents. Platform engineering prevents them.
- The transition is about shifting from reactive (respond to alerts) to proactive (build systems that surface problems before they page).
- The most powerful platform engineering mindset: "what would a good senior SRE reach for at 3am?" Then build that, automate it, and make it available to everyone.

---

### Sequenced learning path toward CNCF contribution

```
Month 1–3   : Go fundamentals → "A Tour of Go" → read krew source
Month 3–6   : OpenTelemetry, Prometheus operator, Cilium/eBPF
Month 6–12  : First CNCF contribution (good first issue → bug report → code PR)
Parallel    : "Designing Distributed Systems" (Brendan Burns)
              "Production Kubernetes" (Josh Rosso et al.)
Security    : CKS → OPA/Kyverno, Falco, network policy
```

### Certifications pathway

```
CKAD  → validates what you already know; fastest pass
CKA   → cluster admin depth; matches real ops work
CKS   → security specialist; bridges observability and security
```

---

## What I'd tell a junior SRE

1. **Read the exit code.** Every good tool tells you if it succeeded. Don't skip it.
2. **Classify before you act.** OOMKill does not always mean "raise the limit". Figure out what kind of OOM it is first.
3. **Name the evidence.** "The service is down" is not a root cause. "Secret `auth-tokens` is missing — 6 pods in FailedMount state, 0 replicas available" is.
4. **Build the causation chain.** When you find a problem, ask "what does this affect next?" until you reach the user-visible symptom.
5. **Make the fix reversible.** Always know how to undo what you're about to do before you do it.
6. **Write it down.** The incident you investigate and don't document will happen again. The one you document becomes a playbook.
