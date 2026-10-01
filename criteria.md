# Acceptance criteria — FitFindr

Five criteria that say what "working" means for this agent, written in unit 3
**before** any results existed.

An acceptance criterion names a target: a number, a count, a rate, or something
a person could plainly observe. *"The agent handles errors"* is an opinion.
*"When search returns nothing, the agent stops before calling the second tool,
in 5 of 5 tries"* is a criterion.

Under each one, write a sentence or two on **why that target** and not a
stricter one. A reason that says something about your tools, your loop, or the
data earns credit; *"80% seemed reasonable"* does not.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

**Two are written for you. You write three.**

---

## 1. A matching query completes all three tools

Given a query that matches at least one listing, the agent completes all three
tool calls and returns a fit card — in at least 4 of 5 tries.

**Why this target:**
I chose 4 of 5 because the search tool relies on keyword matching and some query phrasings may not find a result even when one exists.

---

## 2. An impossible query stops before the second tool

Given a query that matches no listings, the agent stops before calling
`suggest_outfit` and returns a message naming what to change — 5 of 5 tries.

**Why this target:**
The no-results path is deterministic and does not depend on a model, so it should behave correctly every time.

---

## 3. Something about state

The listing ID in `session["selected_item"]` matches the listing ID in `session["search_results"][0]` — in 5 of 5 test runs. Observable by comparing `session["selected_item"]["id"]` against `session["search_results"][0]["id"]` in the returned session dict.

**Why this target:**
Passing state correctly between tools is a core requirement of the agent. If the IDs do not match, the agent may generate recommendations for the wrong item. Because this is a direct dict lookup with no model involved, a single failure means the wiring is broken, not unlucky, so 5 of 5 is the right bar.



---

## 4. Something about the fit card

For any listing input, the fit card includes the listing's title, its price, and its platform — each exactly once — in at least 4 of 5 runs.

**Why this target:**
The prompt explicitly instructs the model to include all three. I target 4 of 5 rather than 5 of 5 because the model occasionally omits or paraphrases the platform name, and that variation is a property of the model output rather than a bug in the tool.



---

## 5. Your choice

When a query includes a price ceiling (e.g. "under $25"), every listing in `session["search_results"]` has a price ≤ that ceiling — in 5 of 5 runs.

**Why this target:**
Price filtering is deterministic arithmetic with no model involvement, so a single failure means the filter is broken rather than unlucky. A bug here would let over-budget results through silently, and the agent would appear to work while violating the user's stated constraint on every run.



---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 4 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath it, like this:

         ## 4. Something about the fit card

         The fit card is different every time.

         **Why this target:** ...

         > **Revised in unit 4:** For 5 different items, the 5 fit cards share
         > no opening sentence.
         >
         > **Why revised:** "different" wasn't checkable — two cards that
         > differed by one word still counted. The new version is something I
         > can actually score.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "I said the empty search stops it 5 of 5 times, but I got 3 of 5,
            so 3 of 5 is more realistic."

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.
     ───────────────────────────────────────────────────────────────────────── -->
