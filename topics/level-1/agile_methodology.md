### Topic: Agile Methodology — Level 1

> **Related:** [Level 1 — Code Repository](./code_repository.md) · [Level 2 — CI/CD](../level-2/ci_cd.md)

**Why it matters (Karat angle)**
Citi runs Agile (mostly Scrum). Interviewers ask this to see if you've actually worked in sprints, participated in ceremonies, and understand velocity — or if you just say "we do Agile" and mean nothing specific. As a senior, you're expected to know how to estimate, unblock yourself, and run a sprint review.

**Core concept**

Agile is a **set of principles** (the Agile Manifesto, 2001) that prioritise:

| Agile values | Over |
|-------------|------|
| **Individuals and interactions** | Processes and tools |
| **Working software** | Comprehensive documentation |
| **Customer collaboration** | Contract negotiation |
| **Responding to change** | Following a plan |

Agile is NOT a methodology — it's a mindset. **Scrum** and **Kanban** are methodologies that implement Agile principles.

**Scrum vs Kanban:**

| Dimension | Scrum | Kanban |
|-----------|-------|--------|
| Time-boxed? | ✅ Sprints (1–4 weeks) | ❌ Continuous flow |
| Roles | Product Owner, Scrum Master, Dev Team | No prescribed roles |
| Ceremonies | Sprint Planning, Daily Standup, Review, Retro | Optional meetings |
| Work limit | Sprint backlog (committed items) | WIP (work-in-progress) limits per column |
| Best for | Feature delivery in cadence | Support/ops, continuous delivery |
| Used at Citi | Most product teams | Ops, support, some DevOps teams |

**Scrum ceremonies:**

| Ceremony | When | Duration (2-week sprint) | Purpose |
|----------|------|--------------------------|---------|
| **Sprint Planning** | Start of sprint | 2–4 hours | Select backlog items, break into tasks, estimate |
| **Daily Standup** | Every day | 15 min max | What I did, what I'll do, blockers |
| **Sprint Review** | End of sprint | 1–2 hours | Demo working software to stakeholders |
| **Sprint Retrospective** | After review | 1–1.5 hours | What went well, what to improve, action items |
| **Backlog Refinement** | Mid-sprint | 1 hour | Clarify upcoming stories, estimate, split large items |

**Scrum artifacts:**

| Artifact | What it is |
|----------|-----------|
| **Product Backlog** | Prioritised list of all work (features, bugs, tech debt) — owned by PO |
| **Sprint Backlog** | Subset committed for this sprint — owned by dev team |
| **Increment** | Working software delivered at sprint end — must be potentially shippable |
| **Definition of Done (DoD)** | Checklist: code reviewed, tests passing, deployed to staging, docs updated |

**Estimation techniques:**

| Method | How | Precision |
|--------|-----|-----------|
| **Story points** (Fibonacci: 1, 2, 3, 5, 8, 13) | Relative complexity, not hours | Low (by design) |
| **T-shirt sizing** (S, M, L, XL) | Quick rough estimate | Very low |
| **Planning Poker** | Team votes simultaneously, discuss outliers | Medium |
| **Time-based** (hours/days) | Direct time estimate | Misleadingly precise |

**Mental model:** Story points measure **complexity + uncertainty + effort**, not hours. A 5-point story isn't "5 hours" — it's "about twice as hard as a 3-pointer." Velocity (average points per sprint) is the team's delivery rate.

**Working example — a sprint board:**

```
| Backlog       | To Do         | In Progress   | In Review     | Done          |
|---------------|---------------|---------------|---------------|---------------|
| Payment audit | Add fraud     |               | Fix timeout   | Account CRUD  |
| Batch reports | check (5 pts) |               | bug (3 pts)   | API (8 pts)   |
| Cache layer   |               |               |               | Unit tests    |
|               |               |               |               | (3 pts)       |
```

**User story format:**

```
As a [role],
I want [feature],
So that [business value].

Acceptance criteria:
- Given [context], when [action], then [expected result]
- Given [edge case], when [action], then [expected handling]
```

**Example:**
```
As a bank teller,
I want to transfer funds between accounts,
So that customers can move money without visiting a branch.

Acceptance criteria:
- Given sufficient balance, when transfer is initiated, then debit source and credit target atomically.
- Given insufficient balance, when transfer is initiated, then reject with a clear error message.
- Given the target account doesn't exist, when transfer is initiated, then return 404.
```

**Agile metrics interviewers might ask about:**

| Metric | What it measures |
|--------|-----------------|
| **Velocity** | Average story points completed per sprint |
| **Burndown chart** | Work remaining vs time in sprint |
| **Cycle time** | Time from "In Progress" to "Done" for one item |
| **Lead time** | Time from backlog to production |
| **Cumulative flow** | WIP across all stages over time (Kanban) |

**What to say in the interview (4-beat answer)**
1. **Definition:** Agile is a set of principles prioritising working software, collaboration, and adaptability. Scrum implements it with time-boxed sprints, defined roles (PO, SM, Dev Team), and ceremonies (planning, standup, review, retro).
2. **Why/when:** Scrum gives predictable delivery cadence and continuous feedback. We plan in 2-week sprints, estimate with story points (Fibonacci), and track velocity to forecast delivery.
3. **Example:** In my last engagement, we ran 2-week sprints. Sprint planning: PO prioritised, team estimated with Planning Poker. Daily standups flagged blockers early. Sprint review: demo to stakeholders. Retro: identified and acted on one improvement per sprint.
4. **Gotcha/tradeoff:** Agile doesn't mean "no planning" — it means "plan frequently in small increments." Teams that skip refinement end up with poorly scoped stories and blown sprints. Velocity is a planning tool, not a performance metric — using it to pressure developers undermines trust.

**Common pitfalls**
- Treating story points as hours — they measure relative complexity, not time.
- Skipping retrospectives — you lose the continuous improvement loop.
- Sprint scope creep — adding stories mid-sprint destroys predictability. New work goes to the backlog.
- 30-minute "standups" — it's a status sync, not a problem-solving meeting. Take discussions offline.
- No Definition of Done — "done" means different things to different people; code review, tests, deployment must be explicit.
- Using velocity to compare teams — different teams have different baselines; only compare a team to itself.

**Self-check question**
Your team's velocity is 40 points per sprint. The Product Owner wants to commit 55 points this sprint because of a deadline. What do you do, and how do you communicate the risk?
