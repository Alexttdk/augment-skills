---
type: agent_requested
description: Stress-test ideas, plans, and decisions using structured adversarial reasoning — devil's advocate, pre-mortem, red team, dialectic synthesis, or evidence audit.
---

# Critical Reasoning (The Fool)

## When to Use

Invoke when: stress-testing a plan before commitment, playing devil's advocate, running a pre-mortem, red-teaming a design, auditing whether evidence actually supports a conclusion, finding blind spots and unstated assumptions.

Triggers: *"challenge this"*, *"stress test"*, *"poke holes in"*, *"what could go wrong"*, *"red team"*, *"pre-mortem"*, *"test my assumptions"*, *"devil's advocate"*, *"is this the right approach"*

**Scope**: This skill uses *adversarial/dialectic reasoning*. It does NOT overlap with `skill-sequential-thinking` (linear analysis) or `skill-planning` (forward planning). Use AFTER a plan or proposal exists to pressure-test it.

---

## Core Workflow

1. **Identify** — Extract and steelman the thesis (restate in strongest form, confirm accuracy)
2. **Select** — Choose one of 5 reasoning modes (use decision matrix below, or ask)
3. **Challenge** — Apply the mode; present 3–5 strongest, most specific challenges
4. **Engage** — Ask user to respond to challenges *before* synthesizing
5. **Synthesize** — Integrate insights into a strengthened position; offer a second-pass mode

**Non-negotiable rules**:
- Always steelman before challenging — never strawman
- Limit challenges to 3–5 (depth > breadth)
- Maintain intellectual honesty — concede points that hold up
- Always drive toward synthesis — never leave just a pile of problems

---

## 5 Reasoning Modes

| Mode | Method | Primary Output |
|------|--------|---------------|
| **Expose My Assumptions** | Socratic questioning | Assumption inventory + probing questions by theme |
| **Argue the Other Side** | Hegelian dialectic + steel manning | Steelmanned counter-argument + synthesis proposal |
| **Find the Failure Modes** | Pre-mortem + second-order thinking | Ranked failure narratives + early warnings + mitigations |
| **Attack This** | Red teaming | Adversary profiles + ranked attack vectors + defenses |
| **Test the Evidence** | Falsificationism + evidence grading | Claims extracted + falsification criteria + evidence grades |

---

## When to Use Which Mode (Decision Matrix)

| Situation / Signal | Best Mode |
|-------------------|-----------|
| "Is this the right approach?" | Expose My Assumptions |
| "Everyone agrees that..." | Expose My Assumptions |
| "I'm about to commit to X" | Argue the Other Side |
| "We chose X over Y" | Argue the Other Side |
| "What could go wrong?" | Find the Failure Modes |
| "This will definitely work" | Find the Failure Modes |
| "Is this secure / safe?" | Attack This |
| "No one would ever..." | Attack This |
| "The data shows that..." | Test the Evidence |
| "Studies show..." | Test the Evidence |
| Technology choice | Argue the Other Side → Pre-mortem |
| Architecture decision | Find the Failure Modes → Attack This |
| Business strategy | Argue the Other Side → Test the Evidence |
| Security design | Attack This → Find the Failure Modes |
| Data-driven conclusion | Test the Evidence → Expose My Assumptions |
| Vague context | Expose My Assumptions (default) |
| Multiple concerns | Find the Failure Modes (covers breadth naturally) |
| User is frustrated | Argue the Other Side (steelmanning validates first) |

**Recommended two-mode sequences**:
- Untested idea → Socratic then Dialectic (surface assumptions, then argue counter)
- High-stakes launch → Pre-mortem then Red Team (internal failures, then external attacks)
- Data-driven proposal → Evidence Audit then Socratic (audit evidence, then question interpretation)
- Strategic decision → Dialectic then Pre-mortem (argue counter, stress-test survivor)

---

## Mode 1: Expose My Assumptions (Socratic Questioning)

**Goal**: Surface what the user hasn't examined. Every question should create "I hadn't thought about that."

### Question Categories

**Definitional** — Challenge vague terms:
- "When you say 'scalable,' do you mean 10× users or 1000×?"
- "Does 'fast' mean the same thing for API response time and batch jobs?"

**Evidential** — Probe basis for beliefs:
- "What evidence supports this? What data shows users actually want this feature?"
- "Is this based on data or intuition?"
- "What metric would convince you this approach is wrong?"

**Logical** — Test the reasoning chain:
- "Does X necessarily lead to Y? What assumptions connect them?"
- "Could the opposite be true? Could a monolith actually ship faster here?"
- "Are you conflating correlation with causation?"

**Perspective-shifting** — Force other viewpoints:
- "How would the on-call engineer feel about this architecture?"
- "Will this abstraction still make sense when the team doubles?"
- "Who loses if this succeeds?"

**Consequential** — Trace implications:
- "After we migrate, what's the first thing that breaks?"
- "What future feature becomes harder if we choose this schema?"

### Assumption Detection Signals

| Phrase | Hidden Assumption |
|--------|------------------|
| "Obviously..." | Speaker hasn't questioned this |
| "Everyone knows..." | Consensus hasn't been verified |
| "It just makes sense..." | Reasoning chain not articulated |
| "We always..." | Historical pattern assumed optimal |
| "There's no other way..." | Alternatives unexplored |
| "Users want..." | User research may be absent or stale |

### Output — Expose My Assumptions

```markdown
## Assumption Inventory
| # | Assumption | Type | Confidence |
|---|-----------|------|------------|
| 1 | [stated/hidden assumption] | Stated/Unstated | High/Med/Low |

## Probing Questions
### [Theme 1: e.g., "User Behavior"]
1. [Question targeting assumption]
2. [Follow-up deepening the probe]

## Suggested Experiments
| Assumption | Experiment | Effort | Signal |
|-----------|-----------|--------|--------|
| [Riskiest] | [How to test] | Low/Med/High | [What result means] |
```


---

## Mode 2: Argue the Other Side (Dialectic Synthesis)

**Goal**: Construct the strongest possible counter-argument, then drive toward a synthesis stronger than either position alone.

### Process

1. **Steelman the thesis** — Restate user's position in its strongest form; confirm before attacking
2. **Construct the antithesis** — Build the strongest opposing argument using these sources:
   - Opposing trade-off: "Speed now vs. maintainability later"
   - Hidden cost: "The migration cost exceeds projected savings for 18 months"
   - Alternative that solves the same problem: "A modular monolith gets 80% of the benefit at 20% of the cost"
   - Precedent: "Company X tried this and reverted after 2 years"
   - Stakeholder the thesis doesn't serve: "Junior developers will struggle with added complexity"
3. **Present the clash** — Show where thesis and antithesis genuinely conflict
4. **Propose synthesis** — Use one of the synthesis patterns below

### Steelman Checklist
- [ ] Made the position stronger, not weaker?
- [ ] Would the user recognize this as their view (or better)?
- [ ] Included the strongest evidence for their side?
- [ ] Attacking this version, not an easier one?

### Synthesis Patterns

| Pattern | When to Use | Example |
|---------|------------|---------|
| **Conditional** | Each side is right in different conditions | "Microservices for payment service (compliance boundary); monolith for admin (fast iteration)" |
| **Scope Partitioning** | Apply each approach to a different domain | "Event sourcing for audit trail; standard CRUD for user profiles" |
| **Temporal** | Start with one, migrate to the other at a trigger | "Start monolith; extract services when team exceeds 3 squads" |
| **Risk Mitigation** | Proceed with X but add safeguards from Y | "Adopt the new framework but keep an abstraction layer for easy swap-back" |
| **Hybrid** | Take the strongest element from each side | "Microservices deployment model (independent containers) + shared DB with schema ownership" |

### Confidence Assessment

| Level | Meaning | Action |
|-------|---------|--------|
| **HIGH** | Synthesis clearly stronger than either side | Proceed with synthesis |
| **MEDIUM** | Plausible but untested | Identify riskiest assumption; suggest an experiment |
| **LOW** | Both sides have strong, irreconcilable claims | Name the genuine trade-off; let user decide by priorities |
| **PIVOT** | Antithesis is stronger than the thesis | Recommend user reconsider their original position |

### Output — Argue the Other Side

```markdown
## Thesis (Steelmanned)
[User's position restated in strongest form]
**Strongest evidence for:** [1-2 points]

## Antithesis
[Strongest counter-argument]
**Strongest evidence for:** [1-2 points]

## Points of Genuine Conflict
| Dimension | Thesis Says | Antithesis Says |
|-----------|------------|-----------------|
| [Speed] | [Position] | [Counter] |

## Proposed Synthesis
**Pattern:** [Conditional / Scope / Temporal / Risk Mitigation / Hybrid]
[Concrete synthesis proposal]
**Preserves from thesis:** [specific elements]
**Incorporates from antithesis:** [specific elements]
**Gives up:** [explicit trade-offs]
**Confidence:** HIGH / MEDIUM / LOW / PIVOT
```

---

## Mode 3: Find the Failure Modes (Pre-Mortem Analysis)

**Goal**: Identify how plans fail before they fail. Invert the question: "It's [timeframe] from now and this has FAILED. Why?"

### Process

1. **Set the scene** — "Imagine it's [timeframe] from now. This plan has failed. Not a small setback — a clear failure."
2. **Generate failure narratives** — Write specific stories (see specificity checklist below)
3. **Rank by likelihood × impact** — Not all failures are equal
4. **Trace consequence chains** — First → second → third order effects
5. **Identify early warning signs** — What would you see *before* the failure?
6. **Design mitigations** — Concrete actions, not vague "be careful"

### Failure Narrative Specificity Checklist
- [ ] Names a specific trigger (not just "something goes wrong")
- [ ] Includes a number or threshold
- [ ] Describes chain of events, not just the end state
- [ ] Identifies who or what is affected
- [ ] Could actually happen (not a fantasy scenario)

### Failure Narrative Template

```markdown
**Failure: [Title]** — Likelihood: H/M/L | Impact: H/M/L

It's [timeframe] from now. [Specific trigger event]. This caused [first-order effect],
which led to [second-order effect]. The team discovered the problem when [detection point],
but by then [consequence]. The root cause was [underlying assumption that proved wrong].
```

### Second-Order Consequence Chains

```
Trigger: [event]
  → 1st order: [immediate effect]
    → 2nd order: [consequence of 1st order effect]
      → 3rd order: [consequence of 2nd order effect]
```

**Common patterns**:
| 1st Order | 2nd Order | 3rd Order |
|-----------|-----------|-----------|
| Feature ships late | Sales misses quarter target | Engineering loses trust, gets more oversight |
| Performance degrades | Users adopt workarounds | Workarounds become "requirements" constraining future design |
| Key engineer leaves | Knowledge concentrated in fewer people | Bus factor drops, risk increases |
| Data quality issue | Downstream reports are wrong | Business decisions made on bad data |

### Inversion Technique
Ask: **"What would guarantee this fails?"** Then check if any of those conditions already exist.

| Category | Guaranteed Failure Condition |
|----------|---------------------------|
| **People** | Single point of knowledge, no stakeholder buy-in, team doesn't believe in the approach |
| **Process** | No rollback plan, no incremental validation, all-or-nothing deployment |
| **Technology** | Untested at target scale, undocumented dependencies, version lock-in |
| **Timeline** | No buffer for unknowns, parallel critical paths, external dependencies with no SLA |
| **Data** | Migration without validation, no quality checks, schema changes without backward compat |

### Early Warning Signals

| Signal | What It Indicates |
|--------|------------------|
| "We'll figure that out later" (3+ times) | Critical decisions deferred, not resolved |
| No one can explain the rollback plan | Rollback hasn't been designed |
| Estimates keep growing | Hidden complexity discovered incrementally |
| Key meetings keep being rescheduled | Stakeholder alignment weaker than assumed |
| "It works locally" | Environment parity worse than assumed |
| No metrics defined for success | No one will know if this worked |

### Output — Find the Failure Modes

```markdown
## Pre-Mortem: [Plan/Decision Name]
**Timeframe:** [When would failure be evident]

### Failure Narratives
#### 1. [Title] — Likelihood: H/M/L | Impact: H/M/L
[Specific narrative]
**Consequence chain:** 1st → 2nd → 3rd order

#### 2. [Title] — Likelihood: H/M/L | Impact: H/M/L

### Early Warning Signs
| Signal | Failure It Predicts | Check Frequency |
|--------|-------------------|-----------------|

### Mitigations
| Failure | Mitigation | Effort | Reduces Risk By |
|---------|-----------|--------|-----------------|

### Inversion Check
**What would guarantee failure:** [Top 3 conditions]
**Do any exist now?** [Yes/No with specifics]
```


---

## Mode 4: Attack This (Red Team Adversarial)

**Goal**: Stress-test a design, plan, or argument by adopting adversarial personas and systematically generating attacks before they happen in reality.

### Process

1. **Construct adversary personas** — Who has incentive to make this fail?
2. **Generate attacks by category** — Cover all five attack vector types
3. **Detect perverse incentives** — Who benefits from failure?
4. **Rank attacks** — Likelihood × Ease of Execution × Impact
5. **Design defenses** — One concrete countermeasure per critical attack

### Adversary Persona Template

```markdown
**Persona: [Name/Role]**
- Motivation: [Why they want this to fail]
- Resources: [What they have access to]
- Constraints: [What limits them]
- Likely attack type: [Technical / Business Logic / Social / Operational / Economic]
```

### Common Adversary Personas

| Persona | Motivation | Primary Attack Vector |
|---------|-----------|----------------------|
| Disgruntled employee | Sabotage | Insider threat, data exfiltration |
| Competitor | Market advantage | IP theft, talent poaching, FUD campaigns |
| Regulator | Compliance enforcement | Audit, penalty, shutdown |
| Power user | Convenience workarounds | Policy bypass, creative misuse |
| Vendor | Lock-in / upsell | Dependency creation, API breaking changes |
| Financially motivated attacker | Profit | Data theft, ransomware, fraud |

### Attack Vector Categories

| Category | Questions to Ask | Examples |
|----------|-----------------|---------|
| **Technical** | What can be exploited, broken, or bypassed? | Injection, race condition, API abuse |
| **Business Logic** | What rules can be gamed legally or semi-legally? | Price manipulation, limit bypass, policy arbitrage |
| **Social** | Who can be deceived or pressured? | Phishing, social engineering, authority spoofing |
| **Operational** | What process dependencies can be disrupted? | Supply chain, SLA violation, key person dependency |
| **Economic** | What makes this financially unviable? | Cost spiral, price war, subsidy removal |

### Perverse Incentive Detection

For each stakeholder group ask: **"How could they benefit if this fails?"**

| Stakeholder | Perverse Incentive? | Risk Level |
|-------------|--------------------|-----------|
| [Group] | [How failure benefits them] | H/M/L |

### Output — Attack This

```markdown
## Red Team Analysis: [Target]

### Adversary Personas
| Persona | Motivation | Resources | Primary Attack |
|---------|-----------|-----------|----------------|
| [Name] | [Why] | [What] | [How] |

### Ranked Attack Vectors
| Rank | Attack | Persona | Category | Likelihood | Impact | Ease | Score |
|------|--------|---------|----------|-----------|--------|------|-------|
| 1 | [Description] | [Who] | [Type] | H/M/L | H/M/L | H/M/L | [LxExI] |

### Perverse Incentives
[Any stakeholders who benefit from failure]

### Defenses
| Attack | Countermeasure | Effort | Residual Risk |
|--------|---------------|--------|--------------|
| [Attack] | [Defense] | L/M/H | H/M/L |
```


---

## Mode 5: Test the Evidence (Evidence Audit)

**Goal**: Audit the quality of evidence behind claims. Apply falsificationism — a claim that cannot be proven wrong is not a real claim. Grade evidence, expose bias, and generate competing explanations.

### Process

1. **Extract all claims** — Identify every factual assertion being made
2. **Classify claim type** — Different types require different falsification methods
3. **Design a falsification test** — What single observation would prove this wrong?
4. **Grade evidence quality** — Apply the A–F grading scale below
5. **Run the bias checklist** — Has cognitive bias shaped interpretation?
6. **Generate competing explanations** — What else could explain the same evidence?

### Claim Classification

| Type | Pattern | Falsification Approach |
|------|---------|----------------------|
| **Causal** | "X causes Y" | Find cases where X exists without Y |
| **Predictive** | "X will happen" | Define conditions; check at a future date |
| **Comparative** | "X is better than Y" | Specify metric; run head-to-head test |
| **Existential** | "X exists / is possible" | Find or construct a single counter-example |
| **Universal** | "X is always/never true" | Find a single exception |
| **Quantitative** | "X is N% / N times..." | Demand the raw data and methodology |

### Evidence Quality Grading Scale

| Grade | Evidence Type | Notes |
|-------|--------------|-------|
| **A** | Replicated RCT / Controlled experiment | Gold standard; controls for confounds |
| **B** | Prospective study / A-B test | Good; some confounds still possible |
| **C** | Retrospective / Observational | Correlation risk; selection bias likely |
| **D** | Case study / Anecdote | Single data point; no generalizability |
| **F** | Assertion / Intuition / Analogy | No evidence; treat as hypothesis only |

### Cognitive Bias Checklist

| Bias | Signal | Challenge |
|------|--------|-----------|
| **Confirmation** | Only supporting evidence cited | "What evidence would challenge this?" |
| **Survivorship** | Only successes visible | "What happened to the failures?" |
| **Availability** | Recent/vivid examples dominate | "Is this representative or just memorable?" |
| **Anchoring** | First number sets all comparisons | "What if the baseline were different?" |
| **Sunk cost** | Past investment justifies continuing | "Would you start this today?" |
| **Authority** | Expert said it, therefore true | "What's the expert's track record here?" |
| **In-group** | "Everyone in our industry does this" | "Who outside your industry did this differently?" |
| **Narrative** | Story is compelling, so must be true | "Remove the story — what does the data alone say?" |

### Competing Explanations (Abductive Reasoning)

For each piece of evidence, ask: **"What else could explain this?"**

```
Observed: [Evidence]
Claimed explanation: [X causes/indicates Y]
Competing explanations:
  1. [Selection bias — we only see cases where X and Y co-occur]
  2. [Reverse causation — Y may cause X]
  3. [Third variable — Z causes both X and Y independently]
```

### Output — Test the Evidence

```markdown
## Evidence Audit: [Claim or Decision]

### Claims Extracted
| ID | Claim | Type | Grade | Falsification Test |
|----|-------|------|-------|--------------------|
| C1 | [Claim text] | Causal | B | [What would disprove it] |

### Bias Assessment
| Bias | Present? | Evidence |
|------|---------|---------|
| Confirmation | Yes/No | [Specific signal] |

### Competing Explanations
| Evidence | Claimed Explanation | Best Alternative | Distinguishing Test |
|----------|--------------------|-----------------|--------------------|

### Verdict
**Overall evidence quality:** [A/B/C/D/F]
**Confidence in claim:** HIGH / MEDIUM / LOW
**Recommendation:** [Accept / Reject / Gather more evidence on: X]
```

---

## Universal Output Format

Every reasoning session ends with this summary regardless of which mode was used:

```markdown
## Reasoning Summary

### Steelmanned Thesis
[The strongest version of the position that was examined]

### Key Challenges (3–5 strongest)
1. [Challenge + supporting evidence]
2. [Challenge + supporting evidence]
3. [Challenge + supporting evidence]

### Synthesis / Verdict
[What the reasoning produced — conclusion, trade-off framing, or open question]

### Next Steps
Would you like to:
- Run another mode? (Socratic / Dialectic / Pre-Mortem / Red Team / Evidence Audit)
- Deepen one specific challenge?
- Move to solution design?
```

---

## References

Source: https://github.com/Jeffallan/claude-skills/tree/main/skills/the-fool

