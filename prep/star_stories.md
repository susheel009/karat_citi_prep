# STAR Stories — Interview Bank

**Source material:** personal fraud detection system experience at TD (verbal notes + Q&A sessions with Claude) + any future additions from `fraud_experience.yaml`.

**STAR format:**
- **S — Situation:** one sentence. What was the context / business problem?
- **T — Task:** one sentence. What was the specific outcome needed?
- **A — Action:** 3–5 sentences. What *I* did (not "we"). Include specific technical decisions.
- **R — Result:** measurable outcome + the lesson / what it signals about me.

**How to use this file:**
1. Each story answers **multiple interview questions** (see "Probe areas this answers").
2. Rehearse aloud — 2 minutes max per story. Record on phone, play back.
3. Tag with topic areas so you can quickly find the right story for a question.
4. Enrich via Q&A rounds — Claude asks, I answer, Claude structures and adds here.

---

## Story 1: Multi-lane Kafka fraud detection pipeline (system design angle)

**Status:** _skeleton — needs enrichment via Q&A_

- **S:** Worked on TD Bank's fraud detection platform, which needed to process real-time events (logins, OTPs, wire transfers, monetary transactions) across multiple business channels.
- **T:** Design an intake architecture that could handle heterogeneous event types, validate them against a common schema, and route them to the right decisioning path without blocking.
- **A:** _[TBD — answer: why 4 swim lanes specifically? What drove the split? Who decided what's "monetary" vs "non-monetary"? Your specific role in the design?]_
- **R:** _[TBD — throughput? latency targets met? operational wins?]_

**Probe areas this answers:**
- "Tell me about a complex distributed system you've worked on"
- "How did you handle heterogeneous data?"
- "Walk me through an architecture you designed"
- "Why did you choose Kafka for this?"

**Tags:** `kafka` `schema-registry` `event-driven` `system-design` `fraud`

---

## Story 2: FICO REST integration with connection pooling and throttling

**Status:** _skeleton_

- **S:** Decisioning service had to call FICO (external third-party) over REST for every fraud check — thousands of requests per minute, hard latency SLA.
- **T:** Ensure connection stability to FICO under load without starving the decisioning service or overwhelming FICO's two-server cluster.
- **A:** _[TBD — specifics: what library/pool? pool size reasoning? throttle threshold? client-side timeout values? How did you measure?]_
- **R:** _[TBD — connection drops reduced? throughput? incident count?]_

**Probe areas this answers:**
- "How do you handle downstream service failures?"
- "Describe an integration with a third-party system"
- "How did you decide on a connection pool size?"
- "What happens if the third-party is slow?"

**Tags:** `rest` `connection-pool` `throttling` `sla` `resilience`

---

## Story 3: Retry + dead-letter queue (DSAP) + response handler

**Status:** _skeleton_

- **S:** FICO occasionally timed out, returned 5xx, or was fully unreachable. Losing a fraud decision event was not acceptable.
- **T:** Guarantee eventual delivery of every fraud decision request, even under partial outage, without silently retrying forever.
- **A:** _[TBD — retry strategy: 3 attempts with exponential backoff (multiplier? base delay?). On final failure, publish to DSAP (dead-letter). Response handler service consumes DSAP and retries with a longer interval. What were the multiplier / base delay values? How did you pick them?]_
- **R:** _[TBD — % of events recovered? zero-data-loss claim?]_

**Probe areas this answers:**
- "Tell me about a time you designed for failure"
- "How do you handle transient errors?"
- "What's a dead-letter queue and why use one?"
- "How did you decide retry parameters?"
- "How did you avoid infinite retry loops?"

**Tags:** `retry` `backoff` `dead-letter-queue` `resilience` `circuit-breaker`

---

## Story 4: Schema registry for heterogeneous data formats

**Status:** _skeleton_

- **S:** Fraud events arrived in multiple formats — Avro, JSON, JSON-encrypted, and occasionally XML — from different producer teams across the bank.
- **T:** Ensure every consumer (intake, enrichment, decisioning) could deserialize and validate events consistently without tight coupling to every producer's format.
- **A:** _[TBD — FSDM (Fraud Structured Data Model/Method) as the canonical schema registered in the schema registry. How was evolution handled? Backwards compatibility? Who owned the schema? Your role specifically?]_
- **R:** _[TBD — e.g., new producer onboarded in days instead of weeks]_

**Probe areas this answers:**
- "How do you manage schemas in event-driven systems?"
- "How do you handle backwards compatibility?"
- "How do independent teams integrate without breaking each other?"

**Tags:** `schema-registry` `avro` `json` `fsdm` `compatibility` `kafka`

---

## Story 5: Enrichment service as a separation-of-concerns boundary

**Status:** _skeleton_

- **S:** Incoming fraud events often lacked customer context — card metadata, account tenure, risk flags — needed for FICO to make an accurate decision.
- **T:** Enrich events with customer data from the data service layer without making intake services themselves coupled to customer data APIs.
- **A:** _[TBD — dedicated enrichment service: input event + output enriched event; who called it; data service layer abstraction (CIF, access card, KIF?); caching if any; what you owned in its design]_
- **R:** _[TBD — decisioning accuracy improvement? latency impact?]_

**Probe areas this answers:**
- "Walk me through a separation-of-concerns decision"
- "How did you avoid the intake service knowing about customer data?"
- "Describe a service boundary you designed"

**Tags:** `service-boundary` `separation-of-concerns` `enrichment` `caching`

---

## Story 6: Handling monetary vs non-monetary vs non-financial events differently

**Status:** _skeleton_

- **S:** Not every fraud event has the same risk profile — a failed login vs a wire transfer need different response times, different decisioning paths, and different operational visibility.
- **T:** Route events to the appropriate swim lane — monetary, non-monetary (OTP entered for a real txn), non-financial (login, OTP generated) — without over-engineering.
- **A:** _[TBD — your role in deciding the lane boundaries; what criteria made a lane decision? How were they physically separated — different topics? Different consumers? Different pod replicas?]_
- **R:** _[TBD — SLA per lane, cost of the split vs single-lane alternative]_

**Probe areas this answers:**
- "How do you reason about system partitioning?"
- "When do you split vs combine services?"
- "Describe a tradeoff you made between simplicity and correctness"

**Tags:** `partitioning` `sla` `swim-lane` `consumer-groups` `kafka`

---

## Open Q&A questions (priority order)

_These are what I (Claude) will ask you, one at a time, to enrich the stories above. Answer freely — I'll structure into STAR and append here._

1. **For Story 1 (multi-lane pipeline):** What drove the decision to split into exactly 4 swim lanes — not 2, not 8? Whose idea was it and how was it validated?
2. **For Story 2 (FICO REST):** What was the pool size, and how did you arrive at that number? What library did you use (Apache HttpClient, OkHttp, built-in Java 11 HttpClient)?
3. **For Story 3 (retry + DSAP):** What were the exact retry values — 3 attempts, what base delay, what multiplier? How did you validate those weren't too aggressive?
4. **For Story 4 (schema registry):** How were schema changes rolled out — who coordinated between producer and consumer teams? Did you ever have a backwards-compat break?
5. **For Story 5 (enrichment):** Was the enrichment service synchronous (blocks intake) or asynchronous? Any caching? What was the hardest part to get right?
6. **For Story 6 (swim lanes):** When an event came in ambiguous — e.g., an OTP generated for a pending wire transfer — which lane did it go to, and how was that decided?

---

## Also useful to capture (non-TD stories — feel free to add)

- [ ] A time you had to learn a new technology fast
- [ ] A disagreement with a colleague / tech lead — how you resolved it
- [ ] A production incident you diagnosed
- [ ] A piece of code you'd refactor if you could go back
- [ ] The best architectural decision you made — or the worst

---

## Format cheat — 4-beat interview delivery

For any STAR story, speak in this rhythm (~90 seconds total):

1. **Set the scene** (10s): "At TD, we had..."
2. **Define the task** (10s): "The specific problem was..."
3. **Describe YOUR action** (50s): "What I did was... First... Then... The tradeoff I considered..."
4. **Wrap with result + signal** (20s): "The result was... — and the lesson I took from it was..."

**Never say "we" when they want to hear about you.** Replace "we" with "I" or "my team did X, and my role was Y."
