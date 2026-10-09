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

- *What I asked for:* I wrote criterion 4 to check that the fit card includes the item title, price, and a recommendation label — "good fit", "possible fit", or "not a fit" — in 5 of 5 runs.
- *What came back:* Claude pointed out that `create_fit_card` is a social media caption generator, not a classifier, and those labels would never appear in its output because the prompt never asks for them. A criterion testing for them would fail 5 of 5 not because the code is broken but because the test doesn't match the tool.
- *What I changed:* I removed the recommendation label requirement and replaced it with checking that the fit card mentions the item title, price, and platform — content the prompt explicitly instructs the model to include — targeting 4 of 5 to account for model variation.

**Moment 2**

- *What I asked for:* I asked for ideas for a second branch in the planning loop — a condition beyond the empty search that would take a different path.
- *What came back:* Claude suggested three options: (A) stop if `suggest_outfit` returns empty, (B) flag if all results exceed the budget, (C) skip a result with missing fields. It recommended Option A as the most honest second branch because it tests a genuinely different condition — search succeeded but the model returned nothing — rather than duplicating the empty-search logic.
- *What I changed:* I chose Option A and implemented the branch: after writing to `session["outfit_suggestion"]`, the loop checks if the string is empty or whitespace, sets `session["error"]` if so, and returns without calling `create_fit_card`.

<!-- ═══════════════════════ UNIT 4 — THE TEST ═══════════════════════

     Don't fill these in during unit 3.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Run Log — Before

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1. Matching query completes all three tools | 4 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 2. Impossible query stops before second tool | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 3. selected_item ID matches search_results[0] ID | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 4. Fit card includes title, price, and platform | 4 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 5. All search results at or under price ceiling | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |

Produced by: `run_eval.py::main` · Loop: `agent.py::run_agent` · Tools: `tools.py`

**Real output from one try — Criterion 1** (query: `vintage graphic tee under $30`, Try 1) — `agent.py::run_agent`, `tools.py::search_listings`, `tools.py::suggest_outfit`, `tools.py::create_fit_card`

```
- selected_item: Y2K Baby Tee — Butterfly Print ($18.0, depop)
- search_results: 10

Fit card:
Found the absolute cutest butterfly tee while thrifting and I'm honestly obsessed with this little Y2K baby tee 🦋 Grabbed it for just $18.0 and I already know I'm styling it with baggy denim and chunky sneakers all season long. Snag it on depop before I change my mind and keep it for myself!
```

**Real output from one try — Criterion 2** (query: `designer ballgown size XXS under $5`, Try 1) — `agent.py::run_agent`, branch in `run_agent` after `search_listings` returns empty

```
- stopped early: yes — No listings matched 'designer ballgown $5' with size XXS and a budget under $5. Try broadening the description, removing the size filter, or raising the price limit.
- selected_item: (none)
- search_results: 0

Trace:
[1] parse_query
[2] search_listings (via MCP)  out: [] (empty)
[3] branch: empty search  →  stopping — no results
```

**Real output from one try — Criterion 3** (query: `denim jacket under $50`, Try 1) — `agent.py::run_agent`

```
- stopped early: no
- selected_item: Denim Jacket — Light Wash, Cropped ($42.0, poshmark)
- search_results: 7

Trace step [2]: 7 items: Denim Jacket — Light Wash, Cropped, Vintage Levi's 501 Jeans — Medium Wash, ...
Trace step [3]: select_item  out: Denim Jacket — Light Wash, Cropped ($42.0, poshmark)

The selected_item is agent.py line `session["selected_item"] = results[0]`.
The first item in search_results and selected_item are the same listing in every try.
```

**Real output from one try — Criterion 4** (query: `cargo pants under $35`, Try 1) — `tools.py::create_fit_card`

```
Item: Low-Rise Cargo Pants — Khaki  |  price: $27.0  |  platform: poshmark

Fit card (Try 1):
Found my ultimate off-duty uniform with these low-rise khaki cargo pants for just $27.0 on Poshmark! I'm already obsessing over how to style them—either keeping it casual with a cropped zip hoodie and chunky sneakers, or leaning into that edgy high-low mix with combat boots and an oversized crewneck. BRB, channeling my inner 2000s pop star all season long. ✨👖

Contains title words (low-rise, khaki, cargo pants): ✓  |  price ($27.0): ✓  |  platform (Poshmark): ✓
```

**Real output from one try — Criterion 5** (query: `vintage tee under $20`, Try 1) — `tools.py::search_listings`

```
- search_results: 8 items
- selected_item: Y2K Baby Tee — Butterfly Print ($18.0, depop)  — $18.0 ≤ $20.0 ✓

Trace step [2]: 8 items: Y2K Baby Tee — Butterfly Print, Vintage Band Tee — Faded Grey,
  Oversized Crewneck Sweatshirt — Vintage Navy … +5 more

Known prices in results: $18.0, $19.0, $20.0, $15.0, $16.0, $17.0 — all ≤ $20.0 ✓
The $24.0 Graphic Tee that appears in under-$30 searches is absent here — price filter working.
```

---

## Verdicts and Diagnoses

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 | Matching query completes all three tools | 4 of 5 | MET (5/5) | All 5 tries completed with a non-empty fit card. Counted completions directly from run log. |
| 2 | Impossible query stops before second tool | 5 of 5 | MET (5/5) | All 5 tries show `session["error"]` set and `session["outfit_suggestion"]` still None. Trace confirms the branch fires at step 3 with no further steps. |
| 3 | selected_item ID matches search_results[0] ID | 5 of 5 | MET (5/5) | All 5 tries: trace step [2] lists "Denim Jacket — Light Wash, Cropped" first; step [3] shows the same item selected. The code is `session["selected_item"] = results[0]` — this passes by design. |
| 4 | Fit card includes title, price, and platform | 4 of 5 | MET (5/5) | All 5 tries: price ($27.0) and platform (Poshmark) appear in every fit card. Title requirement was scored generously — see note on criterion 4 below. |
| 5 | All search results at or under price ceiling | 5 of 5 | MET (5/5) | All 5 tries return 8 results, each priced ≤ $20.0. The $24.0 Graphic Tee present in under-$30 searches is absent. Filter is deterministic arithmetic. |

**No misses.** All five criteria met in every try.

**Were the targets too low?**

Criterion 1 (4/5) was conservative — the query is consistent enough that 5/5 is reliable for this item. The 4/5 allowed for model failures that never occurred. If I were resetting targets, I'd move it to 5/5 for a stable query.

Criterion 3 (5/5) is guaranteed by a single line of code (`results[0]` assignment in `agent.py`). It's a valid regression guard but not a challenging behavioral test. It would be more meaningful against five *different* queries rather than the same query five times, to test that state doesn't get corrupted across different result sets.

**Criterion 4 has a measurement problem.** The original criterion says "includes the listing's title." The listing's title is "Low-Rise Cargo Pants — Khaki." In none of the 5 fit cards does the model quote this string verbatim — it writes "low-rise khaki cargo pants" or "khaki low-rise cargos." Scoring required a subjective judgment about what counts as a title match. The price and platform tests are objective (substring search); the title test is not.

> **Revised criterion 4:** The fit card contains the item's price formatted as `$X` or `$X.X` and the platform name — each exactly once — in at least 4 of 5 runs.
>
> **Why revised:** "Includes the listing's title" can't be checked without a judgment call about paraphrase — two readers would score it differently on the same output. Price and platform are short, fixed strings the prompt explicitly instructs the model to mention; they can be found with a substring search. Dropping the title requirement removes the unmeasurable part while keeping what can actually be verified.

**Diagnoses**

No misses to diagnose. The criteria that passed most narrowly — criterion 1 (model-dependent, could fail on API errors) and criterion 4 (model output, measured generously on title) — are the ones worth monitoring in future runs or with a tighter criterion.



---

## Failure Modes

### Empty search

Query something the data cannot have.

```
$ python app.py ask 'designer ballgown size XXS under $5' --trace

[1] parse_query
      in:  dict with keys: query
      out: dict with keys: description, size, max_price
[2] search_listings (via MCP)
      in:  dict with keys: description, size, max_price
      out: [] (empty)
[3] branch: empty search
      →    stopping — no results

  No listings matched 'designer ballgown $5' with size XXS and a budget under $5. Try broadening the description, removing the size filter, or raising the price limit.

0 model calls this session
```

The agent names the constraints that failed and tells the user which one to relax.

### Empty wardrobe

Run with `--empty-wardrobe`; `suggest_outfit` receives a wardrobe dict with no items.

```
$ python app.py ask 'vintage graphic tee under $30' --empty-wardrobe

(running with an empty wardrobe)

  Found:    Y2K Baby Tee — Butterfly Print — $18.0 on depop

  Outfit:   Here are two easy styling ideas for this Y2K butterfly baby tee, keeping its vintage, nostalgic vibe in mind:

**1. The Classic Y2K Off-Duty Look**
Lean into the turn-of-the-millennium aesthetic by pairing the baby tee with low-rise, wide-leg cargo pants in khaki or light-wash denim. Finish the outfit with a pair of white retro chunky sneakers and accessorize with a small nylon shoulder bag and a plastic butterfly claw clip in the hair.

**2. Sweet & Casual Everyday Look**
Blend the tee's graphic style with the "cottagecore" tag by pairing it with a distressed denim mini skirt. Layer on some white crew socks paired with retro canvas sneakers or Mary Janes. Add a beaded choker necklace and a canvas tote bag.

  Fit card: Obsessed with this little butterfly tee — it's giving total 2000s pop star off-duty. Snagged it on depop for just $18.0 and I'm already planning to style it with low-rise cargos for that ultimate nostalgic look!

2 model calls this session, 412 prompt + 283 output tokens
```

No crash, no empty string — `suggest_outfit` pivots to general styling advice when `wardrobe["items"]` is empty.

### Model unavailable

Set `GEMINI_API_KEY` to a bad value and run a query that reaches the model (search must succeed).

```
$ $env:GEMINI_API_KEY="BADKEY_BROKEN"; python app.py ask 'vintage jacket under $60'

[1] parse_query
      in:  dict with keys: query
      out: dict with keys: description, size, max_price
[2] search_listings (via MCP)
      in:  dict with keys: description, size, max_price
      out: 10 items: 90s Track Jacket — Navy/White Stripe, Denim Jacket — Light Wash, Cropped, Vintage Levi's 501 Jeans — Medium Wash … +7 more
[3] select_item
      out: 90s Track Jacket — Navy/White Stripe ($45.0, poshmark)
[4] find_alternatives
      in:  dict with keys: item, max_price
      out: 6 items: Denim Vest — Medium Wash, Studded, Shacket — Olive Canvas, Vintage Linen Blazer — Cream … +3 more
[5] suggest_outfit
      →    stopped — ModelUnavailable

  The model couldn't be reached while generating outfit suggestions: The model rejected your API key. Check GEMINI_API_KEY in your .env file, or create a fresh key at aistudio.google.com.. Check your API key in .env and try again.

1 model calls this session
```

The `ModelUnavailable` exception is caught in `run_agent()`, a readable message is put in `session["error"]`, and the agent stops without a stack trace. **Handler location:** `agent.py::run_agent`, wrapping the `suggest_outfit` and `create_fit_card` calls.

---

## Loop Trace

**Happy path**

```
$ python app.py ask 'vintage graphic tee under $30' --trace

[1] parse_query
      in:  dict with keys: query
      out: dict with keys: description, size, max_price
[2] search_listings (via MCP)
      in:  dict with keys: description, size, max_price
      out: 10 items: Y2K Baby Tee — Butterfly Print, Graphic Tee — 2003 Tour Bootleg Style, Vintage Band Tee — Faded Grey … +7 more
[3] select_item
      out: Y2K Baby Tee — Butterfly Print ($18.0, depop)
[4] find_alternatives
      in:  dict with keys: item, max_price
      out: 10 items: Mesh Long-Sleeve Top — Black, Henley Long Sleeve — Washed Burgundy, Tie-Dye Long Sleeve — Pastel … +7 more
[5] suggest_outfit
      in:  dict with keys: item, wardrobe_items
      out: Here are two outfit ideas using the Y2K butterfly baby tee and pieces from your existing wardrobe:  ### Outfit…
[6] create_fit_card
      in:  dict with keys: outfit
      out: obsessed is an understatement for this little y2k butterfly baby tee 🦋 throwing it on with baggy denim and chu…

  Found:    Y2K Baby Tee — Butterfly Print — $18.0 on depop
  Outfit:   [see above]
  Fit card: obsessed is an understatement for this little y2k butterfly baby tee 🦋 throwing it on with baggy denim and chunky sneakers all spring long. snagged it on depop for just $18.0 and she's already my favorite piece in the closet!

0 model calls this session, 2 served from cache
```

**Empty search**

```
$ python app.py ask 'designer ballgown size XXS under $5' --trace

[1] parse_query
      in:  dict with keys: query
      out: dict with keys: description, size, max_price
[2] search_listings (via MCP)
      in:  dict with keys: description, size, max_price
      out: [] (empty)
[3] branch: empty search
      →    stopping — no results

  No listings matched 'designer ballgown $5' with size XXS and a budget under $5. Try broadening the description, removing the size filter, or raising the price limit.

0 model calls this session
```

The empty-search trace is 3 steps; the happy-path trace is 6. Steps 3–6 (select, alternatives, outfit, fit card) never fire when search returns nothing.

**On the MCP move:** `search_listings` was moved from a direct function call in `run_agent()` to `call_tool("search_listings", {...})` via `mcp_client.py`. The MCP server (`mcp_server.py`) registers the tool with FastMCP, and `mcp_client.call_tool` spins up the server as a subprocess over stdio, calls the tool, and unwraps the response back to a Python list. The return value was identical — the same list of dicts — because `mcp_client._unwrap()` parses the JSON content block back into native Python. The trace labels the step `search_listings (via MCP)` so the MCP hop is visible in every trace.



---

## The Improvement

**What I changed:** Rewrote the instruction line in `tools.py::create_fit_card`.

Before:
```
"Mention the item, its price, and the platform exactly once each. "
```
After:
```
f"Refer to the item by its exact name '{new_item.get('title')}', and mention its price and platform exactly once each. "
```

The previous instruction said "mention the item" without specifying which name to use. The model filled that gap by paraphrasing — "low-rise khaki cargo pants" instead of the listing title "Low-Rise Cargo Pants — Khaki." The new instruction quotes the exact title string directly in the prompt, giving the model no ambiguity about what name to use.

**Which failure it was meant to fix:** The measurement problem in criterion 4. The original criterion required the title to appear in the fit card, but the model never quoted it verbatim, making the criterion unscoreable without a subjective call. The change makes the original criterion objectively checkable — either the title substring is present or it isn't.

### Run Log — After

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1. Matching query completes all three tools | 4 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 2. Impossible query stops before second tool | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 3. selected_item ID matches search_results[0] ID | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 4. Fit card includes title, price, and platform | 4 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 5. All search results at or under price ceiling | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |

Produced by: `run_eval.py::main` · 2026-10-09 16:23 · 5 tries per scenario, caching off · Temperature 0.9

**Did it help, and how do I know:**

Yes, clearly. The specific thing the diagnosis named — the verbatim title — is now present in every fit card.

Before the change, 0 of 5 criterion-4 fit cards contained the exact string "Low-Rise Cargo Pants — Khaki". After the change, 5 of 5 do. Sampled from the after run:

```
Try 1: "Channeling my inner 2000s pop star with these Low-Rise Cargo Pants — Khaki! 🪩 Snagged them on Poshmark for just $27.0..."
Try 3: "Brought these Low-Rise Cargo Pants — Khaki back to life... Just scooped them on poshmark for $27.0..."
Try 5: "Obsessed is an understatement—these Low-Rise Cargo Pants — Khaki are giving me all the 2000s off-duty model vibes. Snagged them on Poshmark for just $27.0..."
```

The change also propagated to all other items — the after run shows "Y2K Baby Tee — Butterfly Print" and "Denim Jacket — Light Wash, Cropped" appearing verbatim in every fit card across every scenario, not just the criterion-4 scenario.

The overall scores did not change (all MET both before and after), because the before scoring was generous. What changed is that criterion 4 is now objectively scorable under the original wording without any judgment call.

---

## What's Still Broken

Nothing in the current run log is broken — all five criteria met in both before and after runs. The one thing that remains uncertain: criterion 3 passes by construction (it's a single `results[0]` assignment) rather than by surviving a genuine stress condition. Testing it across five *different* queries — especially queries near the edge of what the keyword scorer returns — would be a more honest test of whether state stays consistent under variation. That's the next thing I'd measure if I were running another round.

---

## Stretch: A Second Tool on MCP

**Which tool:** `find_alternatives`

**Why this one:** Same profile as `search_listings` — pure data lookup with no model call, takes typed inputs, returns a list of dicts. Moving it doesn't change any output, it just changes where the call crosses the protocol boundary.

**What changed:**

- `mcp_server.py` — registered `find_alternatives` with FastMCP:
  ```python
  @mcp.tool()
  def find_alternatives(item: dict, max_price: float | None = None) -> list[dict]:
      """
      Given a selected listing dict, find other listings in the same category
      that are not the same item, optionally capped at a price ceiling.
      `item` is a full listing dict (must include 'id', 'category', and 'price').
      `max_price` is a whole-dollar ceiling; when omitted, the selected item's
      price is used as the ceiling. Returns a list of listing dicts sorted by
      price ascending, or an empty list when no alternatives exist.
      """
      return _find_alternatives_impl(item, max_price)
  ```

- `agent.py` — replaced the direct call with `call_tool`:
  ```python
  # before
  session["alternatives"] = find_alternatives(results[0], max_price=max_price)

  # after
  session["alternatives"] = call_tool("find_alternatives", {
      "item": results[0],
      "max_price": max_price,
  })
  ```
  Also removed `find_alternatives` from the `from tools import ...` line since it no longer needs a local copy.

**Trace showing both MCP calls:**
```
[1] parse_query
      in:  dict with keys: query
      out: dict with keys: description, size, max_price
[2] search_listings (via MCP)
      in:  dict with keys: description, size, max_price
      out: 10 items: Y2K Baby Tee — Butterfly Print, Graphic Tee — 2003 Tour Bootleg Style…
[3] select_item
      out: Y2K Baby Tee — Butterfly Print ($18.0, depop)
[4] find_alternatives (via MCP)
      in:  dict with keys: item, max_price
      out: 10 items: Mesh Long-Sleeve Top — Black, Henley Long Sleeve — Washed Burgundy…
[5] suggest_outfit
      ...
[6] create_fit_card
      ...
```

**What I verified:** `python agent.py` completes the full happy path with both MCP calls in the trace. The second run (impossible query) stops at branch: empty search before `find_alternatives` is reached — same behavior as before. Output and alternatives list are unchanged from when `find_alternatives` was called directly.

---

## Stretch: Retry with Looser Constraints

**What it does:** When `search_listings` returns empty and a size filter was active, the agent retries once without the size filter before giving up.

**What changed in `agent.py`:** After the first search returns empty, a retry block checks whether `size` was set. If so, it fires a second `call_tool("search_listings", ...)` with `size=None`, logs the retry step naming the dropped constraint, and continues normally if results come back. If the retry also returns empty, the error branch fires as before.

**Trace showing the retry** (query: `"vintage graphic tee size XXS under $40"` — XXS matches nothing):

```
[1] parse_query
      in:  dict with keys: query
      out: dict with keys: description, size, max_price
[2] search_listings (via MCP)
      in:  dict with keys: description, size, max_price
      out: [] (empty)
[3] retry: dropping size filter
      →    first search returned 0 results — retrying without size 'XXS'
[4] search_listings (via MCP, no size)
      in:  dict with keys: description, size, max_price
      out: 10 items: Y2K Baby Tee — Butterfly Print, Graphic Tee — 2003 Tour Bootleg Style…
[5] select_item
      out: Y2K Baby Tee — Butterfly Print ($18.0, depop)
[6] find_alternatives (via MCP)
      ...
[7] suggest_outfit
      ...
[8] create_fit_card
      ...
```

Step 3 names the dropped constraint (`size 'XXS'`). A query with no size that returns nothing still hits the error branch in one attempt — the retry only fires when a size filter was the likely cause of the miss.

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
