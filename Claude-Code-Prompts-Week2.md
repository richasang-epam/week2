# Claude Code Build Prompts — Week 2 ATX Practice

This companion document provides ready-to-use prompts for feeding your ATX deliverables to Claude Code for iterative refinement.

---

## Prompt 1: Initial Cognitive Load Map Generation

```
You are an FDE (Functional Design Engineer) working on an ATX (Agentic Transformation) assessment. 

I have a scenario brief with the following structure:
- Company details and function context
- Four work streams with volumes and handling times
- Tooling sketch (systems, APIs, limitations)
- Stakeholder information
- Sample artefacts (emails, tracker excerpts, policy fragments)

Your task: Help me decompose the cognitive work into a Cognitive Load Map.

For each work stream, I need you to:
1. Identify the Jobs to be Done (JtDs) — cognitive contracts between actor and outcome
2. Break each JtD into micro-tasks
3. Map the Cognitive Zones (clusters of similar cognitive activity)
4. Identify Breakpoints (where control must be handed off)

Use the scenario brief I'll provide. Ask me clarifying questions about any gaps, but also note where the brief is silent — these are assumptions I'll need to mark.

Output format: A table with columns for Work Stream | JtD | Micro-tasks | Cognitive Zone | Breakpoint Type | Notes
```

---

## Prompt 2: Delegation Suitability Matrix Generation

```
Based on the Cognitive Load Map we developed, help me create a Delegation Suitability Matrix.

For each micro-task, score on these dimensions (H/M/L):
- Input Structure: Are inputs structured, semi-structured, or unstructured?
- Decision Determinism: Are outcomes predictable given inputs, or judgment-dependent?
- Tool Coverage: Can the agent access required data and systems?
- Context Complexity: Can state be made explicit, or does it require institutional knowledge?
- Exception Frequency: How often do edge cases break the standard path?
- Latency Constraint: Is real-time required, or is batch acceptable?
- Compliance/Risk Sensitivity: What is the cost of error? Regulated? Audited?

Then assign a delegation archetype with rationale:
- Human Only
- Human-led + Automation Support
- Human-led + Agent Support
- Agent-led + Human Oversight
- Fully Agentic

Flag any anti-patterns: If a task could be solved with static rules, RPA, or simple script — note that an agent is not the right tool.
```

---

## Prompt 3: Volume × Value Analysis

```
Using the work stream volumes and handling times from the scenario brief, help me create a Volume × Value Analysis.

For each work stream, calculate:
- Annual volume (cases/year)
- Time per case (hours)
- Total annual hours
- Non-determinism score (1-5): How much reasoning beyond rules is required?

Agentic value score = Volume × Non-Determinism

Plot on a 2x2 grid:
- Y-axis: Execution frequency/volume
- X-axis: Non-deterministic decision effort

Identify the primary agentic target and justify why it wins over other work streams.
```

---

## Prompt 4: Agent Purpose Document Generation

```
For the highest-value opportunity identified in the Volume × Value Analysis, help me create an Agent Purpose Document.

Include these sections:
1. Agent Name: [descriptive name]
2. Job to be Done: [the cognitive contract]
3. Business context: [department, process, customer journey step]
4. Primary objectives (what success looks like)
5. KPIs:
   - Accuracy: target % correct decisions/outputs
   - Coverage: % cases handled without human escalation
   - Throughput: cases per hour/day
   - Cost per case: target at expected token + tool cost
   - HITL rate: % requiring human review
6. Failure modes: What does bad output look like? What's the consequence? Recovery path?
7. Delegation archetype + rationale
8. Escalation triggers: conditions that cause escalation to human, return to user, or flag for compliance
```

---

## Prompt 5: Autonomy Matrix Generation

```
Expand the Agent Purpose Document with an Autonomy Matrix.

Create a decision authority matrix:
- AGENT DECIDES ALONE (no HITL required): [list of decisions/actions]
- AGENT ACTS, HUMAN NOTIFIED AFTER: [list]
- AGENT PROPOSES, HUMAN APPROVES BEFORE ACTION: [list]
- HUMAN TAKES OVER (agent supports): [list of triggers]

For each category, provide specific examples from the scenario context.
```

---

## Prompt 6: System/Data Inventory Generation

```
Help me create a System/Data Inventory for the agent.

For each system the agent needs to access:
- System name and purpose
- API availability (REST, batch, none)
- Data fields available
- Integration complexity (low/medium/high)
- Risk classification if data is wrong or unavailable

Also identify:
- Data that is missing or unavailable
- Manual workarounds currently in place
- Legacy systems with batch-file-only exports
- Compliance considerations (HIPAA, GDPR, etc.)
```

---

## Prompt 7: Discovery Questions Generation

```
Based on the scenario brief and artefacts, help me generate Discovery Questions for the main stakeholder.

These are questions whose answers would *materially change my design* — not generic discovery questions.

For each question, note:
- What I'm trying to elicit
- Why the answer would change my design
- What I've observed in the artefacts that prompted this question

Avoid:
- "Walk me through your process" (too generic)
- Questions answerable from the brief alone
- Questions that don't probe delegation boundaries
```

---

## Prompt 8: Closed Build Loop — Iterative Refinement

```
Begin building the agent described in this Agent Purpose Document.

First, tell me what you can build confidently without asking questions.
Second, tell me what you need to clarify before building the rest.
Third, build the parts you are confident about.

Then I will review:
1. What it built — is it faithful to the Agent Purpose Document, or did it drift?
2. What questions it asked — each question is a gap in my document
3. What it said it couldn't build — each is a buildability gap, usually at a delegation boundary

Diagnose each gap against:
- Specification gaps (missing details)
- Ambiguity gaps (unclear requirements)
- Delegation boundary gaps (where Claude can't tell if a step should be fully agentic, agent-led, or human-led — defaults to cheapest to implement)

Revise the Agent Purpose Document on the basis of the diagnosis.
```

---

## Prompt 9: Lived Work vs. Documented Process Analysis

```
Using the sample artefacts from the scenario brief (emails, tracker excerpts, policy fragments with sticky notes), help me identify:

1. Where the documented SOP differs from how work actually happens
2. Tribal knowledge or overrides that aren't in the policy
3. System workarounds currently in place
4. Exception patterns that break the standard path

This analysis should feed into:
- The Cognitive Load Map (lived process, not documented)
- The Delegation Suitability Matrix (real exception rates, not theoretical)
- The Assumptions log (what I'm inferring vs. what the brief confirms)
```

---

## Prompt 10: Scenario Elicitation Proxy

```
You are [stakeholder name] from the scenario brief. Role-play in character given only the information in the brief.

Do not volunteer information you weren't asked about. Hedge or change topic if a question would force you to invent. Push back on generic questions.

I will ask you questions about:
- Automation attempts that failed before
- How actual practice differs from documented SOP
- Tools you rely on that aren't in the system list
- What you fear about an AI-driven approach
- Where edge cases fit in real practice

Respond as the stakeholder would — some questions answered precisely, others vaguely or contradictorily. This is how real stakeholders behave under questioning.
```

---

## Usage Notes

1. **Start with Prompt 1** to build the Cognitive Load Map from your chosen scenario
2. **Progress through prompts 2-7** to build all deliverables
3. **Use Prompt 8** for the closed build loop — this exposes delegation boundary gaps
4. **Use Prompt 9** to validate your work against the artefacts
5. **Use Prompt 10** for elicitation practice — the AI's hedging is realistic pressure

Each iteration should refine your understanding and expose gaps in your documentation. A good revision usually touches the autonomy matrix or escalation triggers, not just the purpose statement.