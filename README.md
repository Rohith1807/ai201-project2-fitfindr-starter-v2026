# FitFindr

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first** — setup, every
> command, and what to do when something breaks.
>
> Once `python test.py` passes:
>
> ```bash
> python app.py listings --full -n 6      # read the data (Milestone 1)
> python app.py fields                    # what you can filter on
> python app.py ask 'vintage graphic tee under $30'
> ```
>
> All three tools are stubs, so that last command will do nothing useful yet.
> That's the starting position.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     HOW TO USE THIS FILE

     This is your submission. Fill each section in as you finish the milestone
     it belongs to — don't leave it all to the end.

     Unit 3 asks for the first five sections. Unit 4 adds the five below them.
     Leave the unit 4 sections alone until then; they're here so you know
     what's coming.

     Everything is pasted as TEXT. No screenshots, no images, no video links.
     A typed block of output gets full credit; a picture of the same output
     gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 3 — THE BUILD ═══════════════════════ -->

## What This Does

FitFindr takes a plain-language query — like "vintage graphic tee under $30, size M" — and searches a dataset of thrift listings for items that match. If something is found, it picks the best match, calls the model to suggest one or two outfits using pieces from the user's wardrobe, and then generates a short social-media-style caption for the find. If nothing matches the filters, it stops early and tells the user exactly which constraint to relax.

---

## Tool Inventory

<!-- Four lines per tool. This is worth 2 points and it's the single most
     common place students lose them.

     "Returns a list" earns NOTHING. The description has to say what is IN
     the list.

     The empty case isn't optional either — it's the thing your loop branches
     on, and if you don't decide it here you'll discover it as a crash in
     Milestone 5. -->

### `search_listings`

- **What it does:** Filters the local listings dataset by price and size, scores the remaining items by keyword overlap with a description, and returns the top matches sorted by relevance score.
- **Inputs:** `description` (str) — keywords describing the item; `size` (str | None) — size token to filter by, e.g. `"M"` or `"S/M"`, or None to skip; `max_price` (float | None) — maximum price inclusive, or None to skip.
- **Returns:** A list of listing dicts (up to `config.SEARCH_RESULT_LIMIT`), each containing `id`, `title`, `description`, `category`, `style_tags` (list), `size`, `condition`, `price` (float), `colors` (list), `brand` (str or None), and `platform`. Items are ordered best match first by keyword overlap score.
- **When it has nothing:** Returns an empty list `[]`. Never raises, never returns None.

### `suggest_outfit`

- **What it does:** Calls the model to suggest one or two outfits for a thrifted item, either naming specific pieces from the user's wardrobe or giving general styling advice when the wardrobe is empty.
- **Inputs:** `new_item` (dict) — a listing dict for the item being considered; `wardrobe` (dict) — a wardrobe dict with an `items` key holding a list of wardrobe item dicts (may be empty).
- **Returns:** A non-empty string containing outfit suggestions. When the wardrobe is populated, the suggestions name specific pieces from it. When the wardrobe is empty, the suggestions are general styling advice for the item type.
- **When it has nothing:** Not applicable — always returns a non-empty string. An empty wardrobe is handled by pivoting to general advice rather than raising or returning `""`.

### `create_fit_card`

- **What it does:** Calls the model to write a 2–4 sentence social-media-style caption for a thrift find, incorporating the outfit suggestion, item name, price, and platform.
- **Inputs:** `outfit` (str) — the outfit suggestion string from `suggest_outfit()`; `new_item` (dict) — the listing dict for the item.
- **Returns:** A string of 2–4 sentences written as a real social post, mentioning the item name, price, and platform each exactly once.
- **When it has nothing:** If `outfit` is empty or whitespace, returns the string `"No outfit suggestion was available to build a fit card from."` without calling the model.

### `find_alternatives`

- **What it does:** Given the selected listing, finds other listings in the same category that are not the selected item, at or under the same price ceiling, sorted by price ascending.
- **Inputs:** `item` (dict) — the selected listing dict; `max_price` (float | None) — price ceiling, or None to use the selected item's own price as the ceiling.
- **Returns:** A list of listing dicts (up to `config.SEARCH_RESULT_LIMIT`), each containing `id`, `title`, `price`, `condition`, `size`, and `platform`, sorted cheapest first. Does not call the model.
- **When it has nothing:** Returns an empty list `[]`. Never raises, never returns None.

---

## Planning Loop

<!-- Your branch rule, stated as a rule — the condition AND both paths — plus
     the file and function that holds it.

     Like this:
       "If search_listings returns an empty list, put a message in the session
        and stop. Otherwise take the first result and go to suggest_outfit."
        — agent.py::run_agent

     The grader checks your code against what you claim here, so the file and
     function have to be real. -->

**Branch rule (1 — empty search):** If `search_listings` returns an empty list, write a message into `session["error"]` telling the user which constraint to relax (size, price ceiling, or description keywords), and return the session immediately — `suggest_outfit` and `create_fit_card` are never called.

**Branch rule (2 — empty outfit):** If `suggest_outfit` returns an empty or whitespace-only string, write a message into `session["error"]` noting that the item was found but the model returned no styling advice, and return the session immediately — `create_fit_card` is never called. `session["fit_card"]` remains `None`.

In all other cases: take the first search result, call `suggest_outfit`, call `create_fit_card`, and return the completed session.

**Where it lives:** `agent.py::run_agent`

**How the query is parsed:** Regex. Three patterns are applied in sequence to the raw query string: (1) `size[:\s]+(\S+)` captures the size token; (2) `under\s*\$?([\d.]+)` or `\$([\d.]+)` captures the price ceiling as a float; (3) the matched spans are removed from the string and the remaining text is cleaned up with `\s+` normalization to produce the description keyword string.

**What moves through the session:** `session["parsed"]` (dict with `description`, `size`, `max_price`) → `session["search_results"]` (list of listing dicts) → branch 1: `session["error"]` and early return, or → `session["selected_item"]` (first listing dict) → `session["alternatives"]` (list of listing dicts) → `session["outfit_suggestion"]` (str) → branch 2: `session["error"]` and early return, or → `session["fit_card"]` (str).

---

## Sample Run

<!-- Two things go here.

     1. One FULL query and its output, pasted as text.
     2. Your three per-tool terminal tests — the command and what it printed. -->

**One full query**

```
$ python app.py ask 'cargo pant under $40'

  Found:    Low-Rise Cargo Pants — Khaki — $27.0 on poshmark

  Outfit:   Here are 2 outfit ideas utilizing the new low-rise khaki cargo pants and pieces you already own:

**Outfit 1: Off-Duty Y2K Streetwear**
*   **Top:** White ribbed tank top
*   **Outerwear:** Black cropped zip hoodie (worn open or layered)
*   **Shoes:** Chunky white sneakers
*   **Accessories:** Black crossbody bag
*   *Why it works:* The low-rise waist and cargo pockets scream 2000s, and pairing the fitted white tank with the cropped hoodie creates the classic Y2K proportions. Chunky sneakers and the black crossbody tie the whole casual streetlook together. 

**Outfit 2: Grungy Contrast**
*   **Top:** Oversized grey crewneck sweatshirt
*   **Shoes:** Black combat boots
*   **Accessories:** Brown leather belt, Black crossbody bag
*   *Why it works:* Tucking or half-tucking the oversized grey crewneck into the low-rise cargos gives an effortlessly slouchy vibe. Swapping sneakers for black combat boots adds an edgy contrast to the neutral khaki, and the brown belt adds a nice touch of warmth to break up the grey and tan. 

*(Note: Since you already own wide-leg khaki trousers, just keep in mind that the new cargos will give you a much more pocket-heavy, utility-focused silhouette compared to your cleaner trousers!)*

  Fit card: Still not over finding these ultimate Y2K low-rise khaki cargo pants for just $27.0 on Poshmark! Paired them with a cropped hoodie and chunky sneakers for the full off-duty look, but honestly obsessed with how they look dressed down with an oversized crewneck and combat boots too. Such a good utility-heavy vibe for fall!

2 model calls this session, 588 prompt + 369 output tokens

```

### stretch challenge output - FIND ALTERNATIVES
```
=== A query the data can match ===
  found:    Y2K Baby Tee — Butterfly Print — $18.0 on depop
  alts:     Mesh Long-Sleeve Top — Black ($15.0), Henley Long Sleeve — Washed Burgundy ($16.0), Tie-Dye Long Sleeve — Pastel ($17.0)
  outfit:   Here are two outfit ideas using the Y2K butterfly baby tee and pieces from your existing wardrobe:

### Outfit 1: Casual & Balanced Y2K Streetwear
*Balance out the fitted, cropped silhouette of the baby tee with baggy bottoms and chunky footwear.*
* **Top:** Y2K Baby Tee — Butterfly Print
* **Bottoms:** Baggy straight-leg jeans, dark wash
* **Outerwear:** Vintage black denim jacket (worn open for layering)
* **Shoes:** Chunky white sneakers
* **Accessories:** Black crossbody bag

### Outfit 2: Sweet & Edgy Contrast
*Combine the girly, cottagecore-leaning butterfly graphic with tougher black accessories for a classic Y2K mix-and-match look.*
* **Top:** Y2K Baby Tee — Butterfly Print
* **Bottoms:** Wide-leg khaki trousers
* **Outerwear:** Black cropped zip hoodie (tied around the waist or worn loosely over top)
* **Shoes:** Black combat boots
* **Accessories:** Brown leather belt
  fit card: obsessed is an understatement for this little y2k butterfly baby tee 🦋 throwing it on with baggy denim and chunky sneakers all spring long. snagged it on depop for just $18.0 and she’s already my favorite piece in the closet!

=== A query it can't ===
  stopped: No listings matched 'designer ballgown $5' with size XXS and a budget under $5. Try broadening the description, removing the size filter, or raising the price limit.
  fit_card is None — it should still be None here

The second one should stop before the fit card. If both paths look the same,
the branch isn't doing anything yet.

```

### second branch — empty outfit suggestion

To trigger this branch, `suggest_outfit` was temporarily stubbed to return `""`.

```
=== A query the data can match ===
  stopped: The outfit suggestion came back empty. The item was found but the model did not return styling advice — try running again or check your API key.
  fit_card is None — it should still be None here

=== A query it can't ===
  stopped: No listings matched 'designer ballgown $5' with size XXS and a budget under $5. Try broadening the description, removing the size filter, or raising the price limit.
  fit_card is None — it should still be None here
```

The first path stops after `suggest_outfit` — `search_listings` succeeded and `selected_item` is set, but `fit_card` is still `None` because `create_fit_card` was never called. This is a different stop condition from the empty-search branch.


**The three tools, tested one at a time**

```
$ python -c "from tools import search_listings; print(search_listings('graphic tee', max_price=30))"

[{'id': 'lst_002', 'title': 'Y2K Baby Tee — Butterfly Print', 'description': 'Super cute early 2000s baby tee with butterfly graphic. Fitted crop length. Tag says medium but fits like a small.', 'category': 'tops', 'style_tags': ['y2k', 'vintage', 'graphic tee', 'cottagecore'], 'size': 'S/M', 'condition': 'excellent', 'price': 18.0, 'colors': ['white', 'pink', 'purple'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_006', 'title': 'Graphic Tee — 2003 Tour Bootleg Style', 'description': 'Vintage-style bootleg tee with faded graphic. Slightly boxy fit. 100% cotton, soft and worn-in.', 'category': 'tops', 'style_tags': ['graphic tee', 'vintage', 'grunge', 'streetwear', 'band tee'], 'size': 'L', 'condition': 'good', 'price': 24.0, 'colors': ['black'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_017', 'title': 'Mesh Long-Sleeve Top — Black', 'description': 'Sheer black mesh long-sleeve. Great for layering under a graphic tee or over a bralette. Stretchy material, fits true to size.', 'category': 'tops', 'style_tags': ['y2k', 'grunge', 'goth', 'layering'], 'size': 'S/M', 'condition': 'excellent', 'price': 15.0, 'colors': ['black'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_033', 'title': 'Vintage Band Tee — Faded Grey', 'description': 'Faded grey band-style tee with distressed graphic. Crew neck. Fits boxy. Well-loved but no holes or major damage.', 'category': 'tops', 'style_tags': ['vintage', 'grunge', 'band tee', 'graphic tee', 'streetwear'], 'size': 'L', 'condition': 'fair', 'price': 19.0, 'colors': ['grey', 'charcoal'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_011', 'title': 'Low-Rise Cargo Pants — Khaki', 'description': 'Y2K era low-rise cargo pants. Lots of pockets. Khaki color, slightly distressed at the hems. Great for layering with a long tee.', 'category': 'bottoms', 'style_tags': ['y2k', 'cargo', '2000s', 'streetwear'], 'size': 'W29', 'condition': 'fair', 'price': 27.0, 'colors': ['khaki', 'tan'], 'brand': None, 'platform': 'poshmark'}, {'id': 'lst_012', 'title': 'Oversized Crewneck Sweatshirt — Vintage Navy', 'description': 'Perfectly faded navy crewneck. Genuinely vintage — not manufactured distressed. Ribbed cuffs and hem. No graphics, clean.', 'category': 'tops', 'style_tags': ['vintage', 'basics', 'oversized', 'classic'], 'size': 'XL (fits oversized)', 'condition':'good', 'price': 20.0, 'colors': ['navy'], 'brand': None, 'platform': 'thredUp'}, {'id': 'lst_015', 'title': 'Vintage Graphic Hoodie — Faded Black', 'description': 'Faded black pullover hoodie with barely-visible vintage graphic on the chest. Cozy interior. Some pilling but adds to the worn-in look.', 'category': 'tops', 'style_tags': ['vintage', 'grunge', 'graphic', 'streetwear'], 'size': 'L', 'condition': 'fair', 'price': 26.0, 'colors': ['black', 'charcoal'], 'brand': None, 'platform': 'depop'}]

```

```
$ python -c "from tools import suggest_outfit; from utils.data_loader import get_example_wardrobe, load_listings; print(suggest_outfit(load_listings()[0], get_example_wardrobe()))"

Here are two specific outfit ideas using the vintage Levi's 501s and pieces from your current wardrobe:

### Outfit 1: Casual Streetwear
* **Tops:** White ribbed tank top
* **Outerwear:** Vintage black denim jacket 
* **Bottoms:** Vintage Levi's 501 Jeans — Medium Wash
* **Shoes:** Chunky white sneakers
* **Accessories:** Black crossbody bag

**Why it works:** The white ribbed tank tucked into the medium-wash 501s creates a clean, classic base. Throwing the black denim jacket over top adds a cool double-denim contrast, while the chunky white sneakers and black crossbody keep the look comfortable and grounded in effortless streetwear.

---

### Outfit 2: Cozy & Edgy Contrast
* **Tops:** Oversized grey crewneck sweatshirt
* **Bottoms:** Vintage Levi's 501 Jeans — Medium Wash
* **Shoes:** Black combat boots
* **Accessories:** Brown leather belt

**Why it works:** Pairing the straight-leg fit of the 501s with an oversized grey crewneck gives you that relaxed, a sharp, edgy contrast to the soft grey and blue tones.

```

```
$ python -c "from tools import create_fit_card; from utils.data_loader import load_listings; print(create_fit_card('jeans and white sneakers', load_listings()[0]))"

Still kicking myself over finding these vintage Levi's 501 jeans—the medium wash on them is literally perfection. Just dropped them over on my depop for $38, and honestly, they're the ultimate grab-and-go pair for everyday streetwear. Honestly, you can't go wrong styling these with just a crisp white tee and fresh sneakers for that effortless 90s off-duty look. ✨

```

### new tool - 'find_alternatives'
```
-- WITH ALTERNATIVES Suggestion
python -c "from tools import find_alternatives; from utils.data_loader import load_listings; item = load_listings()[0]; print(find_alternatives(item, max_price=40))"

Slip Dress — Floral, Midi Length', 'description': 'Delicate 90s slip dress in a muted floral print. Midi length, adjustable straps. Light snag on the side seam — not visible when worn.', 'category': 'bottoms', 'style_tags': ['90s', 'vintage', 'feminine', 'floral', 'cottagecore'], 'size': 'M', 'condition': 'good', 'price': 30.0, 'colors': ['ivory', 'dusty pink', 'green'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_037', 'title': 'Straight Leg Black Jeans — Faded', 'description': 'Faded black straight-leg jeans. Sits at the hips, classic fit. Slightly cropped length. No rips, just natural fading.', 'category': 'bottoms', 'style_tags': ['vintage', 'classic', 'grunge', 'denim'], 'size': 'W28', 'condition': 'good', 'price': 30.0, 'colors': ['black', 'faded black'], 'brand': "Levi's", 'platform': 'thredUp'}, {'id': 'lst_005', 'title': 'Corduroy Wide-Leg Pants — Rust', 'description': 'Beautiful rust-colored cords in a wide-leg silhouette. High-waisted. Minor pilling on the seat but otherwise great condition.', 'category': 'bottoms', 'style_tags': ['vintage', 'cottagecore', '70s', 'earth tones'], 'size': 'W28', 'condition': 'good', 'price': 32.0, 'colors': ['rust', 'orange'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_025', 'title': 'Wide-Leg Linen Trousers — Natural', 'description': 'Relaxed wide-leg linen trousers in a natural/undyed color. Drawstring waist. Very breathable. Sold as a large but cut more like a medium.', 'category': 'bottoms', 'style_tags': ['cottagecore', 'minimal', 'linen', 'earth tones', 'summer'], 'size': 'M/L', 'condition': 'excellent', 'price': 34.0, 'colors': ['natural', 'ecru', 'tan'], 'brand': None, 'platform': 'poshmark'}, {'id': 'lst_031', 'title': 'Baggy Carpenter Jeans — Dark Wash', 'description': 'Baggy carpenter jeans with hammer loop on the side. Dark wash. Sits at the waist. Major 90s workwear vibes.', 'category': 'bottoms', 'style_tags': ['90s', 'vintage', 'streetwear', 'baggy', 'workwear'], 'size': 'W32', 'condition': 'good', 'price': 36.0, 'colors': ['dark blue', 'indigo'], 'brand': None, 'platform': 'depop'}]

-- WITHOUT ALTERNATIVE Suggestions

python -c "from tools import find_alternatives; from utils.data_loader import load_listings; item = load_listings()[0]; print(find_alternatives(item, max_price=0.01))"

[] 

```

---

## How I Used AI

<!-- Two specific moments. What you asked, what came back, what you changed.

     "I used Claude to help me code" is not enough.

     "I gave Claude my search_listings spec. It returned None on no match
     instead of an empty list, so I changed it" is the level we want. -->

**Moment 1**

- *What I asked for:*
- *What came back:*
- *What I changed:*

**Moment 2**

- *What I asked for:*
- *What came back:*
- *What I changed:*

<!-- ═══════════════════════ UNIT 4 — THE TEST ═══════════════════════

     Don't fill these in during unit 3.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Run Log — Before

<!-- Five criteria, five tries each, in this exact format.

     Five, because your criteria are written out of five. Mark each try PASS
     or FAIL, count the passes, and read that count against your target — a
     row targeting 4 of 5 with three PASS cells is MISSED (3/5).

     `python run_eval.py --label before` runs everything and writes the table
     into results/. Paste it here and fill in the verdicts. -->

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Real output from one try**, pasted as text, naming the file and function
that produced it:

```

```

---

## Verdicts and Diagnoses

<!-- MET or MISSED per criterion against LAST UNIT's target, plus a sentence on
     how you decided.

     Then, for every miss: which of the four places it happened — a tool, the
     loop's branch, the session, or the model's output — AND the mechanism.

     Not a diagnosis:  "The fit card was bad."
     A diagnosis:      "The fit card criterion missed on 2 of 5 items. Both had
                        an empty brand field. My prompt puts the brand in the
                        first sentence, so the card opened with a blank and read
                        like a fragment. The tool worked; the prompt assumed a
                        field that isn't always there."

     Look for a pattern. Three misses on the same tool is one problem, not
     three. -->

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 |  |  |  |  |
| 2 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |
| 5 |  |  |  |  |

**Diagnoses**



---

## Loop Trace

<!-- One full run, printed step by step, with the MCP call visible in it.

     `python app.py ask '...' --trace` once you've added the trace.step()
     calls in Milestone 2.

     Worth pasting BOTH the happy path and the empty-search path. The empty
     one should be visibly shorter, because it stops. If your two traces are
     the same length, your branch isn't working — and this is the fastest way
     anyone will ever find that out. -->

**Happy path**

```

```

**Empty search**

```

```

**On the MCP move:** <!-- what changed in your code, and whether anything
behaved differently afterwards. If the rewire didn't work, say exactly where it
broke — the error text and the last thing that worked. That earns the point in
full. -->



---

## The Improvement

<!-- What you changed, why your diagnosis pointed at it, and the after-run in
     the same table format. One change, measured properly.

     `python run_eval.py --label after` -->

**What I changed:**

**Which failure it was meant to fix:**

### Run Log — After

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Did it help, and how do I know:**

<!-- If it made things worse, say that. Honestly reported, that earns full
     credit and is more interesting than one that worked. -->



---

## What's Still Broken

<!-- For each criterion still missed: what you'd do, and why you stopped where
     you did. "I ran out of time" is fine if it's true. Pretending nothing is
     left is not. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 3

       [ ] criteria.md has five numbered criteria, each with a target
       [ ] Each criterion has a reason underneath it
       [ ] All five unit 3 sections above have real content
       [ ] Tool Inventory: all three tools, inputs WITH TYPES, a specific
           return value, and the empty case
       [ ] Planning Loop names the branch rule and agent.py::run_agent
       [ ] Sample Run: one full query plus the three per-tool tests, as text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN, you submit the same one
           next unit

     SUBMISSION CHECKLIST — unit 4

       [ ] mcp_server.py exists with one tool registered
           (or a written record of exactly where the rewire broke)
       [ ] Run Log — Before, five criteria, five tries each
       [ ] Real output pasted underneath, naming file and function
       [ ] A verdict on every criterion
       [ ] A diagnosis for every miss, naming a place AND a mechanism
       [ ] Loop Trace, with the MCP call visible in it
       [ ] All three failure modes triggered and handled
       [ ] One improvement, with Run Log — After in the same format
       [ ] What's Still Broken
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository. Your commit history is what
     shows your criteria existed before your results did.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
