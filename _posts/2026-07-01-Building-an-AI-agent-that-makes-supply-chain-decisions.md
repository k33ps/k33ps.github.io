---
layout: post
title: "Building an AI agent that makes supply-chain decisions — the LLM judges, code computes (1/3)"
categories: engineer 
tags: [LLM, Agent, Supply-chain, AI]
published: True
---

> Series: Part 1, Architecture · [Part 2, Grading judgment](2026-07-08-How-to-grade-an-AI's-judgment.md) · [Part 3, What broke](2026-07-15-I-improved-the-search-and-the-agent-got-worse.md)

Electronics manufacturers receive documents like this. A chip maker sends a PCN (Product Change Notice) — a notice that something about a part you use is changing. On the light end: "the marking on the package surface will change." On the heavy end: "this part goes out of production in six months (EOL); last orders accepted through March." The problem is that these documents don't arrive in a readable form. Every manufacturer uses a different format, and the information that actually matters — which parts are affected, when the last-order deadline is, what the recommended substitute is — is scattered between boilerplate and legal language. It's not structured data. It's a document a human has to read and interpret.

When one arrives, a supply-chain specialist's work begins, and it takes hours to days. Roughly in this order. First, read the notice and figure out which parts are hit. The part numbers in the notice often don't match how your internal system writes them, so even this first cross-referencing step is painful. Next, find out which of your products contain the part. The parts list for a product is called a BOM (bill of materials), and the affected part might be in one BOM, in several, or — if you're lucky — in none. Once you know which products are hit, a decision remains.

There are basically four options.

- Last-time-buy: before production ends, buy everything you'll ever need in one order. Capital gets tied up, storage costs money, and if your demand forecast is wrong you end up with too much or too little.
- Replace: swap in a different part. You have to find candidates, and then re-verify cost, compliance, and lead-time constraints for the whole board.
- Redesign: redesign the board itself. The most expensive and slowest option — but if the first two are blocked, it's the only answer.
- Accept risk: do nothing. For something like a marking change with no real impact, this is the correct answer, and overreacting is itself a cost.

No single step here is hard. What makes this work expensive is that it mixes two different kinds of work. One is the domain of **facts**. Which BOMs are hit, does the candidate's lead time fit inside the deadline, does the swap push cost over the ceiling — these questions are answered by lookups and arithmetic. The other is the domain of **judgment**. Is this change actually dangerous, which of the four options is best right now, do you pick the cheaper candidate or the faster one — these questions are not answered by computation alone. And the work never ends: a product has hundreds of parts, and the notices keep coming.

This situation repeats constantly in the field. The data is already there — stock, lead times, substitute lists, all in the system (though plenty of companies don't even have that). But the last stretch — reading that data and turning it into a decision — has always belonged to a person.

So two questions remained. First, can an agent compress this judgment into minutes? LLMs are good at interpreting messy documents and weighing trade-offs, and the facts are already in a database. With the roles divided properly, it looked possible. Second, if you compress it, can you prove the judgment is any good? Whether "recommend a replacement" was the right call can't be settled by comparing strings. The first question can be answered with a demo. The second question took most of the effort in this project, and it takes up most of this series.

So I built BOM Pilot as a proof of concept. (All data is synthetic — fictional parts, vendors, and prices, unrelated to any real product or dataset.)

## What I built

Usage is simple. Paste the raw text of a manufacturer notice. No reformatting, no looking up part numbers first. A few minutes later you get a recommendation — which strategy, why, and the numbers that back it up. If it recommends a replacement, it also tells you which candidate it picked, why, and what the BOM cost becomes after the swap.

<!-- image file: docs/assets/hero.gif — adjust the path to wherever you host it -->
![BOM Pilot hero run — paste a PCN, watch the graph stream, get the final recommendation](/images/2026-07-01-01.gif)

Inside, it's a fixed pipeline. Not "hand everything to the agent" — there are exactly two points where LLM judgment is needed, and plain code handles everything else:

```
PCN text
   ▼
interpreter (LLM)     parse the notice → affected parts, change type, deadline terms
   ▼
blast radius (code)   which BOM lines are actually hit?
   ├─ no impact ────► report "all clear" and stop (the expensive part never runs)
   ▼
context (code)        prefetch the facts: stock, lead times, cost, constraints
   ▼
planner (LLM) ⇄ executor (code tools)   frame the risk, pick a strategy, compare candidates
   ▼
critic (code)         re-verify the chosen strategy against hard constraints
   ├─ passes ───────► recommendation
   └─ violates ─────► send the planner back to replan
```

What each stage does:

- interpreter (LLM): the first judgment point. It turns the raw text scattered between boilerplate and legalese into structured data: affected part numbers, change type (end-of-life? spec change? marking change?), urgency, last-time-buy terms. The contract includes *not inventing information the notice doesn't contain* — if no deadline is stated, that field stays empty.
- blast radius (code): matches the extracted part numbers against the internal catalog (exact match first, then similar spellings), and finds which lines of which BOMs are affected with a database query. This is the "painful cross-referencing" from the intro — and since it requires no judgment, the LLM doesn't do it. If no lines are affected, the pipeline ends here.
- context (code): gathers the facts about the affected BOM up front and hands them to the planner: the line items, each part's stock and lead time, current cost, and the constraints the BOM must satisfy (cost ceiling, compliance requirements, deadline). This keeps the planner from wasting expensive round trips collecting basics.
- planner (LLM) ⇄ executor (code): the second judgment point, and the heart of the system. The planner frames how risky the situation is, picks a strategy, and compares candidates if a replacement is needed. When it needs more facts, it asks for them through tool calls — part search, per-vendor stock and lead-time lookups, recomputing the BOM cost after a swap. The executor actually runs the tools, and every result comes from the database. The planner can only ask; it cannot make numbers up.
- critic (code): the final gate. It re-checks whether the submitted strategy actually satisfies the non-negotiable constraints: is the post-swap cost under the ceiling, does compliance still hold, does the stock arrive before the deadline. On a violation it sends the planner back with the reasons, and the number of replans is capped. Why this gate is code rather than an LLM is the subject of the next section.

The agent's decision maps directly onto the four options from the intro: ltb / replace / redesign / accept_risk. And the final decision isn't prose — it's a structured submission. Along with the strategy, the agent submits the list of numbers it based the decision on (the *basis*), as a tool call. "Lead time 20 weeks, deadline 8 weeks, therefore rejected" — the numbers holding the decision up are captured in machine-readable form. This format becomes the foundation for the grading in Part 2.

How do you show that a system like this is actually judging, not following a script? By giving it different situations and watching it reach different conclusions. Three demo runs do that job.

- Clear (a few seconds). An EOL notice for a part no BOM uses. Blast radius confirms zero impact, the agent reports "no action needed," and stops. The planner — the most expensive stage — never runs, so cost and response time have a structural ceiling. In practice, a large share of incoming notices are irrelevant to your company, and confirming irrelevance is the most frequently repeated task of all.
- Accept (tens of seconds). A marking-change notice with no electrical impact. The part *is* in a real BOM, but nothing about its function changes, so the agent recommends documenting it and does not go hunting for a replacement. Reflexively swapping parts because a notice arrived is itself a cost — knowing when to do nothing is judgment too, and it gets graded.
- Hero (~3 minutes). A messy notice naming two parts. One is in no BOM; the other is in a real BOM, with a last-time-buy deadline attached. A notice like this naturally nudges you toward "buy" — read "last orders accepted through March" and you start reviewing a last order. But the agent checks with a vendor lookup that the last-time-buy stock would arrive in 20 weeks, compares that against the fact that the deadline is 8 weeks, and rejects the option. Then it searches and compares candidates and recommends a replacement that satisfies the cost and compliance constraints. Following the facts when the document's suggestion and the facts point in different directions — that's the point of this demo.

The same pipeline reaches three different conclusions. What made the difference wasn't the prompt — it was the facts in the database.

The implementation stack is standard. LangGraph (a state graph with a replan loop), PostgreSQL 16 (the single store for parts, vendors, and BOMs), Claude (interpreter and planner), Streamlit (UI), Langfuse (per-call token and cost tracking). Catalog search uses PostgreSQL's built-in full-text search, with no embeddings — I wired up vector search early on, and at this scale it added an external API dependency without adding value, so I ripped it out. But the stack is secondary in this project. What matters is the division of labor between the LLM and code.

## The principle — the LLM judges, code computes

LLM agent demos tend to fail the same way: the model is asked to do arithmetic and enforce rules. The result is a slightly wrong number, fluently narrated. A BOM cost total that's off by a few dollars, or a confident "within the ceiling" that isn't. The more fluent the prose, the harder it is to notice.

So the principle that runs through this project is a division of labor: the LLM judges, code computes.

| The LLM decides | Code computes |
|---|---|
| What this notice actually means | Matching part numbers against the catalog |
| How risky the change is, which strategy fits | Finding affected BOM lines (SQL) |
| Weighing trade-offs between candidates | Recomputing BOM cost, re-verifying hard constraints |

The left column is work that doesn't reduce to computation. The right column does reduce to computation — so there's no reason to give it to an LLM. "The planner can only ask; it cannot make numbers up" is this principle in action.

The place where this boundary matters most is the critic. The critic actually started as an LLM — hand it a rubric and ask, "does this recommendation satisfy the constraints?" The problem with that design is that there's no basis for trusting the grade. If an LLM verifies an LLM, you now need a way to verify the verification. So I rewrote the critic as deterministic code. For each strategy, which values it reads and how it decides is fixed in code:

- replace: recompute the post-swap BOM cost with a tool, then check everything — cost ceiling, compliance requirements, lead time vs. deadline, stock vs. required volume, and the replacement part's lifecycle status.
- ltb: read exactly two numbers — the last-time-buy lead time stated in the notice, and the BOM's deadline. If the stock arrives after the deadline, it's a violation.
- accept_risk: look up the part's lifecycle in the database. If the part is on its way out, or the change affects form, fit, or function, it's a violation.
- redesign: always passes. It's the honest escalation — "no valid fix exists; hand this to engineering" — and that beats insisting on a strategy that doesn't work.

However persuasive the model's writing, if the numbers don't hold, it's rejected. The prose is never an input to the verdict.

This principle wasn't the project's starting point. It's a conclusion I reached after getting something badly wrong.

Midway through, I made a structural mistake. The original design had a loop — a LangGraph graph where the planner proposes, the critic verifies, and a rejection sends the planner back to try again. I tore that out and replaced it with a fixed three-step pipeline: health check → candidate search → analysis, called once each, in order. The logic seemed sound at the time: BOM analysis is a fixed-order task, so no loop needed, and simpler is more stable. The code did get simpler.

The problem surfaced quickly. Replanning had become structurally impossible. The heart of crisis-response judgment is finding another answer when the first one is rejected — if the last-time-buy misses the deadline, look for a replacement; if the replacement busts the cost ceiling, look at another candidate. A straight-line pipeline runs once and it's done; there is no motion of going back and trying again. I deleted the pipeline and rebuilt everything on one graph. The critic changed from an LLM rubric to deterministic code in the same rebuild.

Looking back, the essence of the mistake wasn't framework choice — it was boundary-setting. I traded away freedom in the domain of judgment to buy simplicity. "The LLM judges, code computes" is the sentence that crystallized after this reversal: put the LLM only where judgment lives, and make everything around it deterministic and verifiable. The boundary also produces a useful side effect: when judgment is isolated to two points, what you need to grade becomes obvious.

To lead with the outcome: this agent passes 14 of 17 evaluation scenarios and defends all 5 security scenarios. But that number stands on a much harder question than "how many passed" — who decides whether the agent's decision was right, and with what? Part 2 is the story of that grading.

> Series: Part 1, Architecture · [Part 2, Grading judgment](2026-07-08-How-to-grade-an-AI's-judgment.md) · [Part 3, What broke](2026-07-15-I-improved-the-search-and-the-agent-got-worse.md)