# /assess-launch — Research-Grounded Build & Launch Assessment

Produce a full assessment of a product plan aimed at a successful build and launch: technical plan, monetization, go-to-market, feature set (gaps, cuts, additions), with every market/legal claim grounded in a verifiable source.

## Input

A requirements folder (PRD, product notes, schema, tech decisions) plus the launch market/city. Optional: `$ARGUMENTS` naming specific concerns to prioritise.

## Process

### Step 1 — Absorb the plan
Read every requirements doc fully. Note: roles, epics, monetization mechanics, tech stack, timeline, and any proprietary components. Compare docs against each other for drift (e.g. schema behind PRD).

### Step 2 — Dispatch parallel research subagents
Launch 3-5 background subagents, one per independent thread. Standard set:
1. **Competitive landscape**: local + global competitors per feature area; business models, pricing, traction/shutdown signals; end with biggest threat + genuine white space
2. **Pain points + monetization economics**: evidence-ranked pain points; monetization models that worked/failed in this space with real numbers (take rates, SaaS price anchors); the incumbent workflow being displaced
3. **GTM precedents**: city-by-city / niche-first launch case studies, cold-start mechanics, supply acquisition norms, launch-market specifics (neighborhoods, counts, named actors)
4. **Regulatory exposure** (critical for fintech-adjacent or data-heavy features): payments licensing, data protection, sector-specific licences, in-app currency rules; risk table with severity + compliance path

Every subagent prompt must demand: current-date search (no answering from memory), a URL + date per load-bearing claim, primary sources preferred, low-confidence flags.

### Step 3 — Assess the technical plan yourself while agents run
Scope vs timeline realism (compare against how long comparable single-slice products took), stack fitness for the app's heaviest surfaces, schema-vs-PRD drift, internal contradictions in specs (e.g. offline capability vs real-time cross-device checks), concurrency/consistency hotspots, missing epics (payments, moderation, web presence are the usual suspects). Verify time-sensitive tech claims with quick searches.

### Step 4 — Verify
Spawn a fresh-context verifier subagent over the 10-15 load-bearing claims (legal, pricing, competitive, tech facts the recommendations rest on). Verdict per claim: CONFIRMED / REFUTED / PARTIAL / COULD NOT VERIFY, with URL. Fold caveats into the final text.

### Step 5 — Synthesize
Lead with the findings most likely to sink the launch first, ranked. Then: market verdict (where the product wins / is exposed), monetization recommendation with market anchors, GTM sequence recommendation, feature cut/keep/add lists, technical findings, and a "what cuts against this assessment" section (disconfirming evidence + unvalidated claims). Make calls, don't survey options.

### Step 6 — Persist
Write the assessment (with all source links) to a `.md` in the project's documents folder. After the user chooses which recommendations to adopt, record their decisions as an addendum in the same file, including decisions that overrode recommendations (so they are never re-litigated).

## Output

- Assessment in chat, risks-first
- `Launch-Assessment-<date>.md` with sources + decisions addendum
- Follow-up work (doc updates, schema regen, legal drafts) only after the user picks what's in
