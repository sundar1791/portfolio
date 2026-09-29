# London Postcode Finder — an agentic recommender that learns

**One line:** A multi-agent system that turns "what matters to me" into five honest, researched London neighbourhood recommendations, and gets better with every search.

**Live demo:** [LIVE_DEMO_URL] · **Code:** https://github.com/sundar1791/london-postcode-finder · **How it works:** [LIVE_DEMO_URL]/how-it-works

> Role: product owner, architect and builder. Built part-time over ~6 months, planned as 49 user stories across 8 milestones, using Claude Code for implementation and doing the architectural decisions, data work and evals myself.

---

## 1. The problem

Choosing where to live in London is a trade-off problem disguised as a search problem.

Property portals let you filter by price, bedrooms and area. But the decision someone new to London is actually making sounds more like: *"I want to feel safe, I need to get to work easily, I'd like a park nearby, and I can't spend more than X."* Those factors pull against each other. The places that score well on one usually cost you on another. Today, people resolve that by opening fifteen tabs: crime maps, TfL, rent reports, Reddit threads. Then they stitch it together in their head.

Two things make it hard:

- **The trade-offs are personal.** The same district is right for one person and wrong for another.
- **The data is stale, and the reality isn't.** Official datasets are monthly or annual. What residents say this year — a new line, a closed venue, a policing crackdown — lives in news and forums.

**Hypothesis:** if a system can make people state their trade-offs explicitly, and then combine hard data with current qualitative research, it can replace the fifteen-tab process with five recommendations you can defend.

## 2. The experience

- **100 tokens across 5 dimensions:** safety, green space, nightlife, transport, affordability. A token budget, not "rate each 1–5", because a budget *forces* a trade-off. You can't say everything matters most.
- **Optional free-text context** (≤500 characters) for what sliders can't express, e.g. "I'm terrified of crime" or "I need a nursery nearby for my daughter."
- **Transparent agent activity.** You watch the system decide: how it read your context, whether it re-weighted your tokens, what it researched.
- **Five recommendations:** each has a verdict, rationale, an explicit trade-off, and a practical tip.

## 3. Why agentic — and where it deliberately isn't

The most important product decision was **where not to use an LLM.**

| Part of the problem | Nature | Solution |
|---|---|---|
| Scoring 40 districts on crime, parks, nightlife, transport, rent | Deterministic, factual | Plain data pipelines (**scorers**), pre-computed and cached |
| Interpreting ambiguous user context | Judgement | **Orchestrator agent** |
| Needs no dataset covers ("near a nursery") | Open-ended data gathering | **Context sub-agent**, spawned on demand |
| What it's like to live there *now* | Unstructured, current | **Research agents** with web search |
| Turning it all into advice | Reasoning and writing | **Synthesiser agent** |

Scorers are not agents. They are cheap, fast, testable and never hallucinate. Agency is reserved for the four places where judgement actually adds value.

## 4. Architecture

```
Knowledge Loader ─► Orchestrator ─► (Context Sub-Agent, only if needed)
      ─► 5 Scorers in parallel (from cache, <2s)
      ─► Synthesiser Pass 1 (weighted ranking, pure Python)
      ─► 5 Research Agents in parallel (live web search, top 5 only)
      ─► Synthesiser Pass 2 (recommendations + learnings)
      ─► Knowledge Writer ─► Distiller (every N searches)
```

It is built on LangGraph, with shared state flowing through the nodes. Claude Sonnet handles the reasoning nodes and Claude Haiku handles high-volume research. The backend is Python/FastAPI and the data store is Supabase (Postgres).

**Two passes.** Pass 1 is fast and exhaustive: every district, from cached data. Pass 2 is slow and expensive: live web research, but only on the five shortlisted districts. Research cost scales with the shortlist, not with London.

## 5. Key decisions and trade-offs

**1. Cache the data, don't call APIs live.**
Scoring 40 districts live meant 30–60 seconds of flaky public APIs, with Overpass timeouts and police API zeros. I pre-compute all 200 scores (40 districts × 5 dimensions) monthly and serve them from a table, which takes Pass 1 under 2 seconds.
*Trade-off:* data is up to a month old. That's acceptable for crime statistics and park counts, and the freshness gap is exactly what Pass 2 research fills.

**2. The orchestrator can ignore, adjust, or spawn — and when in doubt, it spawns.**
"I love parks" is *emphasis*, so it adjusts the green weighting. "The park should have nice views" is *qualitative*: it doesn't change the maths, it shapes the language. "I need a nursery" is a *new dimension*, so it spawns a sub-agent to fetch data.
*Why the bias toward spawning:* the errors are asymmetric. A false spawn costs latency, but a false weight adjustment silently distorts every result, and nobody notices.

**3. A spawned need is a filter, not a weight.**
"I need a nursery" is a requirement, not a preference. So nursery scores don't get blended into the ranking. Districts with none nearby are removed from the top 10 before the top 5 is chosen.
*Trade-off:* sometimes fewer than 5 districts survive. That's the honest answer, rather than padding the list with districts that fail the requirement.

**4. Research only the shortlist, with the cheaper model.**
In one early day of testing I spent $4.56, almost entirely on Sonnet reading web search results. Switching research agents to Haiku and capping them at two searches each brought a full run to roughly **$0.10–0.20**. Sonnet stays on the two steps users actually feel: interpreting their context and writing their recommendations.

**5. Memory by distillation, not RAG.**
The obvious move was a vector store. I chose not to. RAG earns its keep when knowledge is large and varied and you need to *select* what's relevant. This domain is the opposite: narrow and repetitive (~20–30 recurring intents, 5 fixed dimensions). After hundreds of searches you don't need to retrieve similar examples. You need a few sharp rules. So the system keeps two tiers, the way an LLM handles a long conversation:
- a **distilled brain** of compressed heuristics, always injected
- the **last 10 raw learnings**, specific and recent

*When I'd switch:* multiple cities or many more dimensions would make the domain broad, and then retrieval becomes the right tool.

**6. Degrade gracefully, everywhere.**
If a public API is down, that district scores 0 and the run continues. If saving a learning fails, the user still gets their answer. Background enrichment is never allowed to break the core experience.

**7. Don't fabricate structure.**
The learnings table was designed for four decomposed fields, but the synthesiser produces one narrative sentence. Rather than force it to invent the other three, I store the sentence and leave those fields empty until a later version genuinely produces them.

## 6. Evals: proving it decides *well*, not just that it runs

A pipeline can run perfectly and still make bad decisions. So evals do three separate jobs:

1. **Baseline correctness.** A deterministic harness asserts 15 real-world rank orderings across all five dimensions. For example, N21 (Winchmore Hill) must score safer than WC1A (Holborn), and N4, which borders Finsbury Park, must score greener than the City. All 15 pass. It reads the cache directly, so a failure means *the data* is wrong, not the wrapper code.
2. **Regression detection.** An end-to-end suite runs three canonical journeys (no context, re-weighting, spawn) against real models. It caught a real regression: after migrating to a newer model, the orchestrator started truncating its JSON on emotionally-worded inputs like "I'm terrified of crime". The cause was a larger tokenizer plus thinking tokens eating the output budget. The fix was one line. Without the suite, it would have shipped.
3. **Judgement quality (next).** An LLM-as-judge harness is next. It grades whether the orchestrator's ignore/adjust/spawn calls are *defensible* on 15 designed boundary cases, and whether recommendations reference the user's actual top priority. The point is to turn "it looked fine" into a score I can track.

## 7. Continuous learning

Every search writes 2–3 learnings. The next search reads them before it reasons. Every N searches, a distiller rewrites the brain: it merges, refines and retires heuristics rather than appending.

A real learning the system wrote to itself after one search:

> *"Central London postcodes like SW1A can score deceptively well on safety-weighted models because government security presence lowers crime statistics while making the area unsuitable for normal residential living, so synthesisers should flag population and residential character data alongside crime scores for such districts."*

That's a non-obvious insight. Heavy policing makes Whitehall look safe, but nobody lives there. Once it's distilled, every future search carries that judgement without having to rediscover it. The `/learning` page shows the current brain and its version history.

A related moment: when a user said they were terrified of crime, the synthesiser actively *warned against* its own fifth-ranked result (N4). It cited documented local crime concerns, instead of dressing the ranking up. The system is allowed to disagree with its own maths when the maths misses something.

## 8. What broke, and what I learned

- **Schema drift.** A table designed in month one rejected real data in month five: the columns were NOT NULL and the category names used American spelling. *Lesson:* test the write path end-to-end as soon as both sides exist.
- **Model deprecation mid-build.** The model I built on reached end-of-life. The replacement returned its reasoning in a new format and used more tokens. *Lesson:* model choice is a dependency with a lifecycle, and evals are what make migrations safe.
- **"Invisible" API rules.** The OpenStreetMap API began rejecting requests without an identifying User-Agent. It surfaced as a vague 406 error, and it would have silently zeroed two dimensions on the next monthly refresh. *Lesson:* scheduled jobs fail quietly, so they need the same scrutiny as user paths.
- **Lost work.** A whole story was lost once because it was never pushed. *Lesson:* commit and push the moment something works. That became a hard rule.

## 9. Beyond postcodes

The pattern is general. Put deterministic scorers on trustworthy signals. Use agents only where judgement is needed. Treat hard requirements as filters. Keep a learning loop that compresses experience into rules, and hold evals over all of it. The same shape fits domains like fleet telematics: vehicle-health signals as scorers, an agent interpreting a fleet manager's context, research on the few vehicles that matter, and a system that learns which interventions actually prevented downtime.

## 10. By the numbers

| Metric | Value |
|---|---|
| Districts × dimensions | 40 × 5 (200 cached scores) |
| Pass 1 latency | under 2 s |
| Full run | ~30–60 s (longer when a spawn fires first time) |
| Cost per search | ~$0.10–0.20 |
| Agents / scorers | 4 agent roles, 5 scorers |
| Deterministic eval | 15 / 15 rank-order assertions pass |
| Plan | 49 user stories across 8 milestones |
