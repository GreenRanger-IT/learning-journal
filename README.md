# SRE to Platform Engineer: Lessons from Production

**By Walz Ogundiran** · Senior SRE & Platform Engineer · [github.com/GreenerPlatform](https://github.com/GreenerPlatform) · [LinkedIn](https://www.linkedin.com/in/olawale-philips-ogundiran/)

---

Five years of production SRE work — managing Kubernetes clusters across GCP and Azure,
running enterprise software for global customers, responding to incidents under pressure —
taught me things no certification covers.

This journal documents the principles, patterns, and mental models I've accumulated.
Not theory. Not tutorials. Real lessons from real production systems, written so
junior SREs can skip the mistakes and senior engineers can find the words for what
they already know.

Each entry is designed to stand alone as a blog post, LinkedIn article, or YouTube video.

---

## Table of Contents

1. [Design Thinking for Operational Tools](#1-design-thinking-for-operational-tools)
   - [The dual-layer pattern](#the-dual-layer-pattern-deterministic-cli--ai-reasoning)
   - [Severity as an exit code](#severity-as-an-exit-code)
   - [Classify before you recommend](#classify-before-you-recommend)
   - [Causation chains vs correlation](#causation-chains-vs-correlation)
   - [Building for the on-call reader](#building-for-the-on-call-reader)

2. [Kubernetes in Production](#2-kubernetes-in-production)
   - [Container-level OOM vs pod-level OOM](#container-level-oom-vs-pod-level-oom)
   - [HPA min==max: autoscaling that doesn't scale](#hpa-minmax-autoscaling-that-doesnt-scale)
   - [SLO burn rates: slow burns are the dangerous ones](#slo-burn-rates-slow-burns-are-the-dangerous-ones)
   - [Reading Cloud Monitoring SLO alerts correctly](#reading-cloud-monitoring-slo-alerts-correctly)

3. [Platform Engineering Mindset](#3-platform-engineering-mindset)
   - [SRE solves incidents; platform engineering prevents them](#sre-solves-incidents-platform-engineering-prevents-them)
   - [Cost is a reliability concern](#cost-is-a-reliability-concern)
   - [Security is platform work](#security-is-platform-work)
   - [The proactive shift](#the-proactive-shift-from-reactive-to-preventive)

4. [Unix CLI Engineering](#4-unix-cli-engineering)
   - [Bash portability: the macOS trap](#bash-portability-the-macos-trap)
   - [fd-redirect: buffering without losing state](#fd-redirect-buffering-without-losing-state)
   - [stdlib-only Python: the distribution advantage](#stdlib-only-python-the-distribution-advantage)
   - [Exit codes are a public API](#exit-codes-are-a-public-api)

5. [AI-assisted Operations](#5-ai-assisted-operations)
   - [Where rules end and LLMs begin](#where-rules-end-and-llms-begin)
   - [Confidence as a first-class output](#confidence-as-a-first-class-output)
   - [Incident history as pattern recognition](#incident-history-as-pattern-recognition)

6. [Career: From SRE to Platform Engineering Leadership](#6-career-from-sre-to-platform-engineering-leadership)
   - [The SRE to platform engineer transition](#the-sre-to-platform-engineer-transition)
   - [The learning path to CNCF contribution](#the-learning-path-to-cncf-contribution)
   - [Certifications that matter](#certifications-that-matter)
   - [Ten principles for junior SREs](#ten-principles-for-junior-sres)

---

## 1. Design Thinking for Operational Tools

### The dual-layer pattern: deterministic CLI + AI reasoning

The right architecture for AI-assisted operational tooling is two layers with a clean
contract between them — not one agent that does everything.

**Layer 1 — The CLI tool:** collects data, applies severity rules, emits structured
JSON. No AI. No network calls beyond the cluster. Runs unattended in CI with no one
watching.

**Layer 2 — The reasoning layer:** consumes the JSON, adds correlation across findings,
builds causation chains, assesses blast radius. This is where LLMs belong.

**The contract between them is simple:** the CLI emits JSON; the reasoning layer
consumes it. Neither layer needs to know how the other works internally.

**Why this matters:** Mixing the layers forces you to choose between CI-safe and
intelligent. The CLI must be deterministic — you can't have a non-deterministic gate
in a deployment pipeline. The reasoning layer must be contextual — you can't express
cross-finding correlation in a lookup table. Separating them gives you both.

**Where this pattern applies beyond SRE:** data pipelines (deterministic ingestion +
ML enrichment), code review tools (static analysis + LLM reasoning), testing
frameworks (rule-based assertions + LLM-generated edge cases).

> **Practice:** Take any monitoring tool you use. Ask: what part of its output is
> deterministic? What part requires context to interpret? The deterministic part
> belongs in a CLI. The contextual part belongs in a reasoning layer.

---

### Severity as an exit code

A health checker without a machine-readable exit code is just pretty output.

Exit codes are the API of shell tools. Every script, every CI pipeline, every
orchestrator communicates with tools through exit codes. If your tool exits 0 when
it finds critical problems, every automated system that calls it is blind.

**The pattern:**

```
0 → all clear       (pipeline continues, deploy proceeds)
1 → warnings        (pipeline continues with a warning logged)
2 → critical        (pipeline blocks, deploy is stopped)
```

This is exactly how `grep` works (`0`=found, `1`=not found, `2`=error), how
`test` works, how every good Unix tool works. It is not a new idea — it is the
Unix convention, applied to health checking.

**Mental model:** exit codes are HTTP status codes for shell tools. `0`=200 OK,
`1`=something worth noting (4xx), `2`=something that blocks (5xx).

**Consequence:** once you adopt this, your health checker becomes composable. You
can write `kubectl-sentinel && deploy.sh` and the deploy only runs when the cluster
is healthy. No shell logic needed. The tool does the right thing.

> **Practice:** check the exit codes of every tool in your CI pipeline. You will find
> at least one that exits 0 on failure and silently passes bad state downstream.

---

### Classify before you recommend

"Pod is OOMKilled → raise the memory limit" is wrong advice roughly 40% of the time.

Before prescribing a fix, classify the problem. The classification determines the
action. Getting this wrong wastes time at best and makes things worse at worst.

**OOMKill classification:**

| Class | Signal | Action |
|-------|--------|--------|
| `LIMIT TOO LOW` | runtime before kill < 5min, memory climbing steadily | Raise the limit in config |
| `POSSIBLE LEAK` | runtime hours/days, all replicas fail over time | Request memory profiling — do NOT raise limit without profiling |
| `INPUT TRIGGERED` | crash correlates with specific request pattern or payload size | Fix input handling |

Raising the limit on a leak just delays the next crash by a few hours. The leak fills
the bigger limit and the pod crashes again. You have spent your incident response time
on the wrong action.

**This principle generalises to every finding type:**

- `CrashLoopBackOff` → classify by exit code: 1=app error, 137=OOM kill, 143=SIGTERM
- `ImagePullBackOff` → classify: auth failure vs network vs quota exceeded
- Pending pods → classify: resource pressure vs node affinity vs PVC unavailable
- Probe failure → classify: liveness (pod is being killed) vs readiness (pod is excluded from traffic)

> **Rule:** The P1 action must follow from the classification. "Restart the pod" or
> "raise the limit" without a stated classification are not acceptable incident
> responses.

---

### Causation chains vs correlation

Two findings appearing together does not mean one caused the other.

When an OOMKill and a service availability drop appear in the same incident, the
temptation is to say "OOMKill caused the degradation." That may be true. But you
need to state the mechanism, not just the correlation.

**The Via: pattern:** every link in a causation chain must name the propagation
mechanism:

```
OOMKill (scheduler-service)
  → pod restarted, in-flight requests dropped
    Via: synchronous HTTP — callers received 502 during restart window
  → API availability dropped below SLO threshold
    Via: no retry logic in the calling service
  → burn rate alert fired on 1-hour window
```

If you cannot name the mechanism, the link is inferred — not confirmed. Mark it:

```
OOMKill (scheduler-service)
  ⚑ INFERRED → API latency increase
    Verify: kubectl logs <api-pod> -n <ns> | grep -i "connection refused\|timeout"
```

An inferred link with a verification command is useful. An asserted causal link with
no evidence is dangerous — it sends the engineer down the wrong remediation path.

> **Practice:** next time you're in an incident, write the causation chain before you
> act. If you can't name the mechanism for each link, add `⚑ INFERRED` and write
> the command to verify it. You will be surprised how often "obvious" causation is
> actually inferred.

---

### Building for the on-call reader

Every design decision in an SRE tool should be made as if the person reading the
output has been on call for hours, is stressed, and needs to act in 5 minutes.

This constraint changes everything:

**Report order:** most critical findings first. Not alphabetical. Not the order you
wrote the checks. The order that saves the most time for the engineer who opens the
report in a state of high stress.

**Language:** specific beats vague, always.

> `"FailedMount events detected"` → useless. The engineer still has to go figure out which secret, which pods, what impact.
>
> `"Secret auth-tokens missing — 6 pods in FailedMount, 0 replicas available, service entirely down"` → actionable in one read.

**Commands:** give the exact command. Not "check the pod logs." Give:
```bash
kubectl logs <pod-name> -n <namespace> --previous --tail=50
```

**Classification before recommendation:** a stressed engineer given a recommendation
without a classification will act on it regardless of whether it fits. Classification
first prevents the wrong fix being applied under pressure.

> **Design test:** take the output of any tool you own and ask: "could an engineer
> who has never seen this system before understand what is wrong and what to do in
> 90 seconds?" If not, the tool needs work.

---

## 2. Kubernetes in Production

### Container-level OOM vs pod-level OOM

A pod can have multiple containers with independent memory limits. When a pod gets
OOMKilled, you need to identify which container was killed — not just which pod.

**How to find the affected container:**

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Look at the `Last State` section for each container individually. The container with
`OOMKilled` in Last State and a short runtime before the kill (under 5 minutes) is
the one hitting its limit. Other containers in the same pod may be healthy.

**Why this matters:** the pod name is not the container name. A pod named
`scheduler-service-xyz` might have two containers — the scheduler container with a
210Mi limit that is crashing, and a sidecar with a 2Gi limit that is healthy. You
cannot tell which container is the problem without reading each container's `Last
State` independently.

**The symptom that misleads you:** the pod shows `OOMKilled` in `kubectl get pods`.
The natural assumption is that the pod's memory limit is the issue. But the pod has
no single memory limit — each container has its own. You must describe the pod to
see the per-container limits and Last State.

> **Command reference:**
> ```bash
> # See per-container limits and last state for a pod
> kubectl describe pod <name> -n <ns> | grep -A5 "Limits\|Last State"
> ```

---

### HPA min==max: autoscaling that doesn't scale

A HorizontalPodAutoscaler with `minReplicas == maxReplicas` is functionally inert.

The HPA cannot scale up because it is already at its maximum. It cannot scale down
because it is already at its minimum. The controller is running, watching metrics,
computing desired replicas — and then doing nothing, because the desired count is
always clamped to the one value where min and max meet.

**This is not a bug — it is a misconfiguration.** The HPA is working exactly as
specified. The specification is wrong.

**How to spot it:**

```bash
kubectl get hpa -n <namespace>
# Look for MINPODS == MAXPODS
```

**Why this appears in production:** teams set `min == max` to "lock" a deployment at
a fixed replica count while still having the HPA in place "for later." The HPA
remains, quietly doing nothing, until someone sets a CPU target during an incident
and wonders why the pods aren't scaling.

**The fix:** raise `maxReplicas` to a value above the current replica count, or set
a meaningful `minReplicas` below the current count if scale-down is desired.

> **Audit command:**
> ```bash
> kubectl get hpa -A | awk '$5 == $6 {print "INERT:", $0}'
> # Prints any HPA where MINPODS equals MAXPODS
> ```

---

### SLO burn rates: slow burns are the dangerous ones

A 2x burn rate on a 1-hour window means your error budget is being consumed twice as
fast as it should be. An 8x burn rate on a 24-hour window will exhaust your monthly
error budget in approximately 3.7 days — but it will not trigger a fast-window alert
because the current error rate is low.

**This is the slow burn problem.** The service is not down. Response time is slightly
elevated. Error rate is marginally above normal. No page fires for the on-call
engineer — but by Thursday, the monthly budget is exhausted and every blip for the
rest of the month triggers a page.

**Why the alert window matters:**

| Window | What it catches |
|--------|----------------|
| 1-hour | Fast burns — service is effectively down right now |
| 6-hour | Medium burns — service is significantly degraded |
| 24-hour | Slow burns — service is slightly degraded but the budget is draining |
| 72-hour | Very slow burns — healthy-looking but unsustainable over the month |

Multi-window alerting (fire on 1h AND 24h burn rate simultaneously) gives you
defence in depth: fast fires catch outages, slow fires catch the quiet drains.

> **Key insight:** when you see a burn rate alert, do not look at the current error
> rate. Look at the window error budget consumption. The alert tells you the rate of
> consumption — not the current state.

---

### Reading Cloud Monitoring SLO alerts correctly

GCP Cloud Monitoring SLO alerts fire on `select_slo_burn_rate` — the rate at which
the error budget is being consumed against a specific measurement window. They resolve
automatically when the burn rate drops below the configured threshold.

**What this means operationally:** do not manually resolve these alerts. The alert
will close itself when the service recovers. Manually closing it hides the signal —
if the burn rate picks up again, you want a new alert.

**Parsing the alert title:** GCP SLO alert titles typically embed the service name
and environment. Building a lookup table that maps environment tokens in the title
(e.g., `[PROD]`, `[DEV]`) to namespaces and clusters is worth the investment — it
removes a manual lookup step from every triage.

**The correct question when you receive this alert:** not "is the service down?" but
"how fast is the error budget draining, and will it exhaust before the end of the
measurement period?" The burn rate multiplier gives you the answer: a 3x burn rate
on a 30-day budget drains it in 10 days at the current rate.

---

## 3. Platform Engineering Mindset

### SRE solves incidents; platform engineering prevents them

SRE is reactive by design. You are paged. You investigate. You fix. You write a
postmortem. The goal is to shorten the time between "problem starts" and "problem
resolved."

Platform engineering is proactive by design. You ask: what would a good senior SRE
need mid-incident? Then you build that, automate it, and make it available to everyone —
before the incident.

**The shift in questions:**

| SRE question | Platform engineering question |
|-------------|-------------------------------|
| "Why is this pod crashing?" | "How do we surface pod crashes before they page?" |
| "Which service is causing this?" | "How do we make service dependencies visible in normal operation?" |
| "What's the root cause?" | "How do we build tooling that builds the causation chain automatically?" |
| "How do we fix this incident?" | "How do we make this class of incident self-diagnosing?" |

The platform engineering mindset does not stop investigating incidents. It takes every
investigation and asks: "what would have made this faster, or caught it earlier?" The
answer to that question is the next tool to build.

> **Transition marker:** you have started thinking like a platform engineer when you
> finish an incident postmortem and immediately open a ticket to build the tool that
> would have made the investigation unnecessary.

---

### Cost is a reliability concern

Cloud cost and service reliability are not separate concerns. They are coupled.

An oversized cluster costs money and hides performance problems — services that would
fail on correctly-sized nodes fail silently on an oversized cluster because spare
capacity absorbs the spike. An undersized cluster causes reliability incidents — not
enough CPU or memory, pods are evicted, HPAs cannot scale, nodes go NotReady.

**Cost-reliability tradeoffs appear in every area of platform work:**

- **Node sizing:** too large hides problems; too small causes them
- **Resource requests/limits:** too high wastes capacity and prevents scheduling; too low causes OOM kills and CPU throttling
- **HPA configuration:** `minReplicas` too high wastes cost at idle; `maxReplicas` too low causes ceiling incidents under load
- **Idle infrastructure:** dev environments left running overnight at full size; test clusters not scaled down on weekends

**The platform engineer's position:** cost governance is part of reliability engineering.
Budget alerts, cluster autoscaler configuration, scheduled scale-down for non-production
environments, and resource request right-sizing are as much a platform responsibility
as alerting and dashboards.

> **Key signal:** if your team treats cost spikes as a finance problem and reliability
> incidents as an engineering problem, they are missing the coupling between them.
> The best platform teams treat cost anomalies as early reliability signals.

---

### Security is platform work

Security in Kubernetes is not the security team's problem. It is the platform team's
problem, because the security controls live in the platform layer.

**The controls that matter and who owns them:**

| Control | What it does | Platform's role |
|---------|-------------|----------------|
| RBAC | who can do what in the cluster | design and audit role bindings |
| Pod Security Standards | prevent privileged pods, hostPath mounts, root containers | enforce at namespace level |
| Network Policy | which pods can talk to which | define default-deny and explicit allow |
| Secret hygiene | no stale secrets, least-privilege access | audit and rotate |
| TLS configuration | correct certificate chains, up-to-date cipher suites | own the ingress and service mesh config |

**The compliance angle:** enterprise platforms operate under security standards that
require continuous compliance. A certificate with a name mismatch, an Azure storage
account without soft-delete enabled, a service bus without TLS 1.2 minimum — these
are platform misconfigurations that can trip a compliance gate and take services
offline. Security hygiene is not optional in regulated industries.

**The platform engineer's role in security:** you are not writing exploits or running
penetration tests. You are building the guardrails: the policy enforcement, the audit
tooling, the automated detection of misconfiguration. Security is an operational
concern that the platform team is best positioned to own.

> **Start here:** run `kubectl get clusterrolebindings | grep cluster-admin` in your
> cluster. Count how many non-system subjects have `cluster-admin`. That number
> should be very small. If it isn't, that is a platform problem.

---

### The proactive shift: from reactive to preventive

The highest-leverage thing a platform engineer does is not fix the current incident
faster — it is make the next incident shorter or prevent it entirely.

**Three levels of proactive platform work:**

**Level 1 — Faster detection:** instrumentation that catches problems before users
report them. Prometheus alerts, SLO tracking, cluster health checks on a schedule.

**Level 2 — Automated diagnosis:** tooling that collects the data an engineer would
gather in the first 15 minutes of an incident and presents it structured. The goal
is not to replace the engineer — it is to eliminate the part of the investigation
that is mechanical and predictable.

**Level 3 — Systemic prevention:** changes to the platform that make a class of
incident impossible or self-healing. Automatic secret rotation so stale secrets
cannot cause FailedMount incidents. Resource quota enforcement so one team cannot
starve another. PodDisruptionBudgets so node upgrades cannot take a stateful set
entirely offline.

Most SRE teams operate at Level 1. Platform engineering teams aim for Level 2 and
build toward Level 3.

---

## 4. Unix CLI Engineering

### Bash portability: the macOS trap

macOS ships bash 3.2 — a version from 2007. The reason is legal, not technical:
bash 4+ is licensed under GPLv3 and Apple does not ship GPLv3 software with the OS.

**This means these common bash 4+ features silently fail on macOS:**

```bash
declare -A mymap          # associative arrays — not available
mapfile -t arr < file     # read file into array — not available
echo "${var^^}"           # uppercase expansion — not available
```

**The portable fix for associative arrays:** use parallel indexed arrays.

```bash
# Instead of:
declare -A counts
counts["node"]=3

# Use:
KEYS=("node")
VALUES=(3)
# Access: ${VALUES[0]} corresponds to ${KEYS[0]}
```

**Why this matters for distribution:** if you ship a bash tool and it works perfectly
on your Linux machine but fails silently on a colleague's Mac, you will spend hours
debugging a bash version problem that looks like a logic error. Test on bash 3.2
before shipping.

> **Test command:** `bash --version` — if the output says `3.2`, you are on macOS's
> default shell. All your tools must work in this environment if you want them to be
> truly portable.

---

### fd-redirect: buffering without losing state

When you need to reorder output — print a summary before details that are computed
after the details — the Unix answer is file descriptor redirection.

**The problem:** you want to print a status banner (CRITICAL: 3, WARN: 5) at the
top of your report. But you don't know the CRITICAL count until all checks have run.
If you run checks first and print the banner after, the banner is at the bottom.

**The solution:**

```bash
exec 3>&1           # save stdout to file descriptor 3
exec 1>"$tmpfile"   # redirect stdout to a temp file
set -e

# All output goes to tmpfile. Global counters still update correctly.
run_all_checks

exec 1>&3           # restore stdout
exec 3>&-           # close fd 3

# Now print in the order you want:
print_banner        # uses the counters populated during checks
cat "$tmpfile"      # replay the captured check output
```

**Key insight:** file descriptors are process-level state. Redirecting stdout to a
file does not affect global variable updates. Your counters accumulate correctly even
while output goes to the temp file. This solves the ordering problem without
restructuring your entire check function layout.

> **When to reach for this:** any time you need to print aggregate information
> (counts, totals, summaries) before the details that produce them.

---

### stdlib-only Python for distribution

A Python tool with zero pip dependencies is dramatically easier to distribute.

**No dependencies means:**
- No `requirements.txt` for users to install
- No virtualenv for users to create
- No pip step in CI
- No version conflict between your package and the user's environment
- Works on any machine with Python 3.x — which is essentially every machine

**The stdlib replacements:**

| Common package | stdlib replacement |
|---|---|
| `requests` | `urllib.request` |
| `rich` | ANSI codes directly (`\033[0;31m`) |
| `PyYAML` | `json` (emit JSON instead of YAML) |
| `click` | `argparse` |

**The distribution advantage:** your install instruction is `curl <url> > tool.py && chmod +x tool.py`.
Compare to: `pip install virtualenv && virtualenv venv && source venv/bin/activate && pip install -r requirements.txt`.

The stdlib-only tool wins every time on adoption in environments where you cannot
control what is installed.

> **Rule:** before adding any dependency, ask "does this exist in stdlib?" The answer
> is yes more often than you expect.

---

### Exit codes are a public API

The exit code of a CLI tool is part of its public API. Changing an exit code is a
breaking change — every script that calls your tool and branches on the exit code
will behave incorrectly after the change.

**Document exit codes in the module docstring, not just in `--help`:**

```python
"""
incident-triage: deterministic alert triage tool.

Exit codes:
  0 — triage complete, structured output produced
  1 — namespace not determinable; user input required
  2 — input snapshot missing or schema mismatch
  3 — no sentinel findings match the alert symptom (low confidence)
"""
```

**Treat exit codes as a versioning concern:** if you change what exit code 1 means,
you have broken backwards compatibility. Bump the major version.

**Design your exit codes for composability:**

```bash
incident-triage --alert "payments 503" --json | jq '.confidence'
# exit 0 = full triage, parse the JSON
# exit 3 = low confidence, inspect manually before acting
```

---

## 5. AI-assisted Operations

### Where rules end and LLMs begin

A CLI tool can determine that a pod is OOMKilled. It cannot determine that this
OOMKill caused the API degradation because the crashing service is in the synchronous
request path of the user-facing endpoint.

That requires understanding system architecture — which services call which, what
the failure modes are, what the deployment topology looks like. This context changes
with every sprint. It cannot be expressed in a lookup table.

**The practical dividing line:**

| Use a rule (CLI) | Use an LLM (reasoning layer) |
|-----------------|------------------------------|
| "Is memory usage above the limit?" | "Is this OOM caused by a leak or a limit misconfiguration?" |
| "Is the pod in CrashLoopBackOff?" | "Is this crash related to the deployment that happened 20 minutes ago?" |
| "Is the HPA at maxReplicas?" | "Should maxReplicas be higher given the current traffic pattern?" |
| "Are there recent OOMKill events?" | "Which of these OOMKills is the root cause and which are victims?" |

**The rule:** use deterministic code for what is always true regardless of context.
Use LLMs for what requires reading context to interpret correctly.

A lookup table that maps "OOMKill → cause service degradation" will be wrong in 90%
of cases because the causal relationship depends on system topology that varies per
deployment. An LLM reading alert descriptions, service names, and namespace topology
can reason about it correctly.

---

### Confidence as a first-class output

A triage report that omits its confidence level is dangerous. The engineer reading it
will either over-trust it (act on a wrong root cause) or under-trust it (ignore a
correct diagnosis). Making confidence explicit forces the tool to reason about the
quality of its own evidence.

**A three-level system:**

| Level | When to use it | What it tells the engineer |
|-------|---------------|---------------------------|
| `HIGH` | Direct match between alert symptom and CRITICAL finding in the expected namespace | Act on the recommended fix — evidence is strong |
| `MEDIUM` | Indirect evidence, or correlation without confirmed mechanism | Investigate before acting — the direction is right but verify |
| `LOW` | Alert symptom does not match any cluster finding | Manual investigation required — cluster state collected for context only |

**Always include a one-sentence reason:**

```
Confidence: MEDIUM — OOMKill in the expected namespace matches the alert
symptom, but the causal link to the API degradation is inferred, not confirmed.
```

This single line tells the engineer exactly what to verify before acting.

> **Principle:** a tool that tells you "the root cause is X" with no confidence
> qualifier is making a claim it cannot support. Confidence is not a weakness
> signal — it is an honesty signal.

---

### Incident history as pattern recognition

The most valuable use of PagerDuty incident history in triage is not retrieving
alert metadata — it is pattern recognition.

The questions historical data answers:

- Is this a one-off or a recurring pattern?
- Did a previous fix hold, or is this the same root cause re-emerging?
- Are related services also degrading at the same time (shared root cause)?
- Was there a deployment or configuration change shortly before this alert fired?

**Why this changes the response:**

"4th time in 6 months, last occurrence: 2026-02-14" does not mean "restart and
monitor." It means "the root cause was not fixed in February, March, or whenever
else this fired. This needs an ADO ticket, not a restart."

"First occurrence, no related incidents, change event 12 minutes before alert" means
"start with the change event — the deployment is the most likely trigger."

**Historical context sets the confidence of the causation chain** and determines
whether the P3 (prevent recurrence) action is "monitor and close" or "escalate to
engineering for root cause fix."

---

## 6. Career: From SRE to Platform Engineering Leadership

### The SRE to platform engineer transition

SRE and platform engineering are different orientations toward the same problems.
SRE asks: "how do we respond to this?" Platform engineering asks: "how do we build
systems that respond for us?"

The transition is not a role change — it is a mindset shift that happens gradually as
you accumulate enough incident experience to see the patterns.

**Signs you are making the transition:**

- You finish an incident and immediately think about the tooling that would have made it faster, not just the fix that resolved it
- You write postmortems that produce ADO tickets for tooling work, not just configuration fixes
- You start building things that junior engineers use, not just things that you use
- You think about platforms as products: who are the users, what do they need, how do you reduce their cognitive load

**The highest-leverage transition move:** take one class of incident that you
respond to repeatedly and build the tool that automates the first 15 minutes of
investigation. The tool doesn't need to be clever. It needs to collect the right
data in the right order and present it structured. That is platform engineering.

---

### The learning path to CNCF contribution

A sequenced path from production SRE to CNCF contributor:

```
Now                  Solidify Kubernetes depth: CKAD → CKA → CKS
                     kubectl, Helm, ArgoCD, Istio, Prometheus — know the source code

Month 1–3            Go fundamentals
                     Why: all major CNCF tools are written in Go
                     Start: "A Tour of Go" (tour.golang.org)
                     Then: read the source of krew (small, well-structured CNCF tool)

Month 3–6            CNCF ecosystem depth
                     OpenTelemetry: distributed tracing, metrics, logs
                     Prometheus operator: PrometheusRule CRDs, Alertmanager routing
                     Cilium/eBPF: network policy and observability at the kernel level

Month 6–12           First CNCF contribution
                     Start: documentation issues (labelled good-first-issue)
                     Progress: bug reports with reproduction steps and root cause
                     Progress: small code contributions

Parallel             Architecture foundations
                     "Designing Distributed Systems" — Brendan Burns (co-creator of Kubernetes)
                     "Production Kubernetes" — Josh Rosso et al.
                     CNCF landscape: one day per major category; understand the problem each project solves
```

**Why Go specifically:** kubectl, Helm, Prometheus, Thanos, Kiali, ArgoCD, Cilium —
the entire observability and platform tooling stack you work with daily is written in
Go. Reading source code and eventually contributing requires it.

---

### Certifications that matter

Certifications in Kubernetes are not about the certificate — they are about the
preparation process, which forces you to deeply understand concepts you may have been
using at surface level.

```
CKAD  (Certified Kubernetes Application Developer)
      Validates: workloads, config, networking, observability, storage
      Time: 2 hours, hands-on terminal
      Value: fastest cert to pass; validates what most SREs already do

CKA   (Certified Kubernetes Administrator)
      Validates: cluster architecture, installation, configuration, networking
      Time: 2 hours, hands-on terminal
      Value: forces depth on cluster admin concepts; matches real platform work

CKS   (Certified Kubernetes Security Specialist)
      Requires: active CKA
      Validates: cluster setup, hardening, supply chain, runtime security
      Time: 2 hours, hands-on terminal
      Value: bridges observability and security; high demand in enterprise
```

**Study resource order:**
1. killer.sh practice exams — the closest simulation to the real exam environment
2. "Kubernetes in Action" by Marko Luksa — best book for practitioners (not managers)
3. Kubernetes docs: read the concepts section end to end, not just what you need

---

### Ten principles for junior SREs

These are the things I wish someone had told me on my first on-call shift.

**1. Read the exit code.**
Every good tool tells you whether it succeeded. The exit code is often more reliable
than the output. Check it.

**2. Classify before you act.**
OOMKill does not mean "raise the limit." CrashLoopBackOff does not mean "restart."
Identify the class of problem before choosing the fix.

**3. Name the evidence.**
"The service is down" is not a root cause.
"Secret `auth-tokens` missing — 6 pods in FailedMount state, 0 replicas available" is.
Be specific. Name the resource. State the impact.

**4. Build the causation chain before you act.**
When you find a problem, ask "what does this affect next?" and follow the chain until
you reach the user-visible symptom. Act on the root, not the symptom.

**5. Make the fix reversible.**
Before you apply a fix, know exactly how to undo it. If you can't, find a smaller
fix you can undo first, verify it works, then proceed.

**6. Write it down.**
The incident you investigate without documenting will happen again. The one you
document becomes a playbook. The playbook becomes the tool that prevents it.

**7. Understand the alerting window.**
When you receive a burn rate alert, look at the window — not the current error rate.
The window tells you whether this is an emergency or a slow drain.

**8. The HPA needs both min and max to be right.**
An HPA with minReplicas == maxReplicas does nothing. Check both values whenever you
investigate a capacity incident.

**9. Cost anomalies are reliability signals.**
If a cost spike appears without a corresponding traffic increase, something is
misusing resources. Investigate it as a reliability concern, not a billing concern.

**10. Security misconfigurations are operational incidents waiting to happen.**
A stale secret, a misconfigured certificate, an overly permissive RBAC binding — 
these don't alert until they cause an outage. Treat security hygiene as proactive
reliability work.

---

## About

Built while shipping production SRE tooling for Kubernetes clusters running across GCP
and Azure. Tools referenced in this journal:
[kubectl-sentinel](https://github.com/GreenerPlatform/kubectl-sentinel) ·
[incident-triage](https://github.com/GreenerPlatform/incident-triage)

I also write **[AI Reliability Principles](https://www.linkedin.com/company/ai-reliability-principles/)** —
a series on making AI dependable enough to trust with real operational decisions
([read on Medium](https://olawaleogundiran.medium.com)).

If something here helped you, the best way to say thanks is to open an issue on one
of those repos with a problem you're trying to solve.
