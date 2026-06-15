# FitFindr — planning.md

## Tools

### Tool 1: search_listings

**What it does:** Searches the mock secondhand listings dataset and returns items that match the user's keywords, optional size filter, and optional price ceiling. Results are sorted by relevance score (most keyword matches first).

**Input parameters:**
- `description (str)`: Keywords describing what the user is looking for (e.g., "vintage graphic tee"). Used for keyword matching against title, description, and style_tags.
- `size (str | None)`: Size to filter by (e.g., "M"). Case-insensitive. Pass None to skip size filtering.
- `max_price (float | None)`: Maximum price in dollars (inclusive). Pass None to skip price filtering.

**What it returns:** A list of listing dicts sorted by relevance score, highest first. Each dict contains: id, title, description, category, style_tags (list), size, condition, price (float), colors (list), brand, platform. Returns an empty list `[]` if nothing matches — never raises an exception.

**What happens if it fails or returns nothing:** Returns `[]`. The agent checks for this immediately after calling the tool. If the list is empty, it sets `session["error"]` to a message telling the user what to try differently, and returns early without calling `suggest_outfit` or `create_fit_card`.

---

### Tool 2: suggest_outfit

**What it does:** Given a thrifted item and the user's current wardrobe, calls the Groq LLM to suggest 1–2 complete outfit combinations using the new item and specific pieces from the wardrobe.

**Input parameters:**
- `new_item (dict)`: A listing dict returned by `search_listings` — the item the user is considering buying.
- `wardrobe (dict)`: A wardrobe dict with an `"items"` key containing a list of wardrobe item dicts. Each wardrobe item has: id, name, category, colors (list), style_tags (list), notes.

**What it returns:** A non-empty string with outfit suggestions from the LLM. If the wardrobe is empty, returns general styling advice for the item instead of crashing.

**What happens if it fails or returns nothing:** If `wardrobe["items"]` is empty, the agent builds a different prompt asking for general styling ideas rather than wardrobe-specific combinations — so the tool always returns a useful string. If the LLM call fails, the exception propagates and the agent surfaces it as an error to the user.

---

### Tool 3: create_fit_card

**What it does:** Generates a short 2–3 sentence Instagram/TikTok-style caption for the thrifted outfit. Calls the Groq LLM with a high temperature setting so the output varies each time.

**Input parameters:**
- `outfit (str)`: The outfit suggestion string returned by `suggest_outfit`.
- `new_item (dict)`: The listing dict for the thrifted item — used to include the item name, price, and platform in the caption.

**What it returns:** A 2–3 sentence casual caption string that mentions the item name, price, and platform once each. If `outfit` is empty or whitespace-only, returns a descriptive error string instead of crashing.

**What happens if it fails or returns nothing:** Guards against an empty `outfit` string at the top of the function — returns `"Error: Cannot create a fit card without an outfit suggestion."` rather than raising an exception.

---

## Planning Loop

The agent runs a linear sequence of tool calls, but **branches based on what each tool returns**:

1. Parse the user's query using regex to extract `description`, `size`, and `max_price`. Store in `session["parsed"]`.
2. Call `search_listings(description, size, max_price)`. Store results in `session["search_results"]`.
3. **Branch:** If `search_results` is empty → set `session["error"]` to a helpful message and **return early**. Do NOT call `suggest_outfit` or `create_fit_card`.
4. If results exist → set `session["selected_item"] = results[0]` (top result).
5. Call `suggest_outfit(selected_item, wardrobe)`. Store result in `session["outfit_suggestion"]`.
6. Call `create_fit_card(outfit_suggestion, selected_item)`. Store result in `session["fit_card"]`.
7. Return the completed session.

The key adaptive decision is step 3: the agent only proceeds to `suggest_outfit` if there is something to work with. It never calls all three tools unconditionally.

---

## State Management

All state is stored in a single session dict created by `_new_session(query, wardrobe)` at the start of each interaction. The dict holds:

- `query` — original user input
- `parsed` — extracted description, size, max_price
- `search_results` — full list of matching listing dicts
- `selected_item` — the top result, passed directly into `suggest_outfit`
- `wardrobe` — the user's wardrobe dict, passed into `suggest_outfit`
- `outfit_suggestion` — string from `suggest_outfit`, passed directly into `create_fit_card`
- `fit_card` — final caption string
- `error` — set if the interaction ended early; None on success

No data is re-entered by the user between steps. The item found in step 2 is the same object passed into step 5.

---

## Error Handling

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| `search_listings` | No listings match the query | Sets `session["error"]` to: "No listings found for '[query]'. Try different keywords, a higher price, or remove the size filter." Returns early without calling the other tools. |
| `suggest_outfit` | Wardrobe is empty (`wardrobe["items"]` is `[]`) | Builds a different LLM prompt asking for general styling advice instead of wardrobe-specific outfits. Returns a useful string — never crashes. |
| `create_fit_card` | `outfit` argument is empty or whitespace-only | Returns the string `"Error: Cannot create a fit card without an outfit suggestion."` — does not raise an exception. |

---

## Architecture
```
User query
│
▼
run_agent(query, wardrobe)
│
├─ Step 1: _new_session(query, wardrobe) → session dict
│
├─ Step 2: Parse query with regex → session["parsed"]
│           (extracts description, size, max_price)
│
├─ Step 3: search_listings(description, size, max_price)
│           │
│           ├─ results == [] → session["error"] = "No listings found..."
│           │                  return session  ← EARLY EXIT
│           │
│           └─ results exist → session["search_results"] = results
│                              session["selected_item"] = results[0]
│
├─ Step 5: suggest_outfit(selected_item, wardrobe)
│           │
│           ├─ wardrobe empty → general styling advice prompt
│           └─ wardrobe has items → wardrobe-specific outfit prompt
│           session["outfit_suggestion"] = result
│
├─ Step 6: create_fit_card(outfit_suggestion, selected_item)
│           │
│           ├─ outfit empty → return error string
│           └─ outfit valid → LLM caption prompt (temp=1.2)
│           session["fit_card"] = result
│
└─ Step 7: return session
```
---

## AI Tool Plan

**Milestone 3 — Individual tool implementations:**
I used Claude as an AI tool. I provided the Tool 1 spec block from this planning.md (inputs, return value, failure mode) and asked it to implement `search_listings` using `load_listings()` from `data_loader.py`. I reviewed the generated code against the spec before using it — specifically checking that it filtered by both price and size, scored by keyword overlap, dropped zero-score results, and returned an empty list on no match. I also used Claude to implement `suggest_outfit` and `create_fit_card`, providing the Tool 2 and Tool 3 spec blocks. I revised the `suggest_outfit` output after discovering the wardrobe schema uses `"colors"` (a list) not `"color"` (a string), which caused a KeyError.

**Milestone 4 — Planning loop and state management:**
I used Claude to implement `run_agent()` by providing the Planning Loop and State Management sections of this planning.md along with the Architecture diagram. I verified the generated code checked for an empty `search_results` list before calling `suggest_outfit`, and confirmed that `session["selected_item"]` was passed directly into `suggest_outfit` without re-entry.

---

## A Complete Interaction (Step by Step)

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1:**
Tool: `search_listings`
Input: `description="vintage graphic tee"`, `size=None`, `max_price=30.0`
Why: This is always the first tool — the agent needs a real item before it can suggest an outfit or write a caption.
Output: A list of 2–3 matching listings sorted by relevance. Top result: `{"title": "Faded Band Tee", "price": 22.0, "platform": "Depop", "size": "M", "condition": "Good", ...}`

**Step 2:**
Tool: `suggest_outfit`
Input: `new_item={"title": "Faded Band Tee", ...}`, `wardrobe=<example wardrobe with baggy jeans, chunky sneakers, etc.>`
Why: The item was found — now the agent uses the user's actual wardrobe items to build specific outfit combinations.
Output: `"Pair the Faded Band Tee with your wide-leg jeans and platform Docs for a 90s grunge look. Roll the sleeves once and tuck the front corner for shape."`

**Step 3:**
Tool: `create_fit_card`
Input: `outfit="Pair the Faded Band Tee..."`, `new_item={"title": "Faded Band Tee", "price": 22.0, "platform": "Depop", ...}`
Why: The outfit is ready — the agent generates a shareable caption to complete the interaction.
Output: `"thrifted this faded band tee off depop for $22 and it was literally made for my wide-legs 🖤 full look in my stories"`

**Final output to user:** The top listing details, the outfit suggestion, and the fit card are each displayed in their own panel in the Gradio UI.