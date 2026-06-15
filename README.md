# FitFindr 🛍️

A multi-tool AI agent that helps users find secondhand clothing and figure out how to wear it. Built for CodePath AI201 Project 2.

---

## Tool Inventory

### `search_listings(description: str, size: str | None, max_price: float | None) → list[dict]`
Searches 40 mock secondhand listings by keyword overlap against title, description, and style_tags. Filters by size and price if provided. Returns a list of matching listing dicts sorted by relevance score (highest first). Returns `[]` on no match — never raises an exception.

### `suggest_outfit(new_item: dict, wardrobe: dict) → str`
Calls the Groq LLM (llama-3.3-70b-versatile) to suggest 1–2 complete outfit combinations using the new thrifted item and named pieces from the user's wardrobe. If the wardrobe is empty, returns general styling advice instead of crashing.

### `create_fit_card(outfit: str, new_item: dict) → str`
Calls the Groq LLM at temperature 1.2 to generate a 2–3 sentence Instagram/TikTok caption that mentions the item name, price, and platform. Guards against an empty `outfit` argument — returns an error string instead of raising an exception.

---

## How the Planning Loop Works

`run_agent()` runs a conditional sequence — not a fixed pipeline:

1. Parse the query with regex to extract description, size, and max_price.
2. Call `search_listings`. If it returns `[]`, set an error message and **return early** — `suggest_outfit` and `create_fit_card` are never called.
3. If results exist, take the top result as `selected_item`.
4. Call `suggest_outfit` with the selected item and wardrobe.
5. Call `create_fit_card` with the outfit suggestion and selected item.
6. Return the session dict.

The agent behaves differently depending on what `search_listings` returns. An impossible query like "designer ballgown size XXS under $5" stops at step 2 and gives the user a specific, actionable error message.

---

## State Management

All state lives in a session dict created at the start of each `run_agent()` call. Key fields:

- `session["parsed"]` — extracted description, size, max_price from the query
- `session["search_results"]` — full list of matching listing dicts
- `session["selected_item"]` — top result, passed directly into `suggest_outfit`
- `session["outfit_suggestion"]` — string from `suggest_outfit`, passed directly into `create_fit_card`
- `session["fit_card"]` — final caption string
- `session["error"]` — set on early exit; None on success

No data is re-entered by the user between steps. The item found by `search_listings` is the exact same dict object that flows into `suggest_outfit`.

---

## Error Handling

**`search_listings`:** Returns `[]` on no match. The agent immediately checks this and returns: *"No listings found for '[query]'. Try different keywords, a higher price, or remove the size filter."* Tested by running query `"designer ballgown size XXS under $5"` — confirmed returns `[]` and agent exits early.

**`suggest_outfit`:** Handles empty wardrobe gracefully — checks `wardrobe["items"]` before building the prompt and switches to a general styling prompt if empty. Tested with `get_empty_wardrobe()` — confirmed returns a useful string instead of crashing.

**`create_fit_card`:** Guards against empty `outfit` at the top of the function — returns `"Error: Cannot create a fit card without an outfit suggestion."` instead of raising. Tested by calling `create_fit_card("", item)` directly — confirmed returns error string.

---

## Spec Reflection

**One way planning.md helped:** Writing out the exact field names and types for each tool before coding caught the wardrobe schema issue early — I knew `wardrobe["items"]` was a list of dicts and planned the prompt formatting around that. When I hit the `KeyError: 'color'` bug, I immediately knew to check the schema because I'd already written down that the field was `"colors"` (a list).

**One divergence from the spec:** The planning loop spec described parsing the query in a separate dedicated step before calling `search_listings`. In implementation, I combined the parsing and search setup into a single block using `re.search` and `re.sub` inline rather than a separate parse function. This worked fine for the scope of the project but a production version would benefit from a dedicated LLM-based parser for more complex queries.

---

## AI Usage

**Instance 1 — Implementing `search_listings`:**
I gave Claude the Tool 1 spec block from planning.md (inputs with types, return value description, failure mode) and asked it to implement the function using `load_listings()` from `data_loader.py`. The generated code correctly filtered by price and size and scored by keyword overlap. I reviewed it and revised the scoring logic to also search `style_tags` (not just `title` and `description`), because style tags like "vintage" and "grunge" are important search signals that the initial output missed.

**Instance 2 — Implementing `suggest_outfit`:**
I gave Claude the Tool 2 spec block and the wardrobe schema from `data/wardrobe_schema.json`. It generated code that accessed `w['color']` (singular). I caught this by cross-referencing the schema, which shows `"colors"` as a list. I corrected it to `', '.join(w['colors'])` before running. This is an example where reviewing the generated code against the actual data structure caught a bug before it hit production — and it did still cause a `KeyError` when I ran the app without catching it first, confirming why this review step matters.

---

## Setup

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Create a `.env` file:
```
GROQ_API_KEY=your_key_here
```
Run the app:
```bash
python app.py
```

Open `http://localhost:7860` in your browser.