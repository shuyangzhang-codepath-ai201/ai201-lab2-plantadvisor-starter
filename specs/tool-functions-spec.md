# Spec: Tool Functions

**File:** `tools.py`
**Status:** `get_seasonal_conditions` — Pre-implemented, read through. `lookup_plant` — complete spec fields before implementing.

---

## Purpose

These two functions are the tools the agent can call. They retrieve structured data from the local plant database and seasonal data files and return it to the agent loop, which passes it to the LLM as context for generating a response.

---

## Function 1: `lookup_plant()`

### Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `plant_name` | `str` | The plant name as entered by the user or chosen by the LLM — may be any casing, common name, scientific name, or alias |

**Output:** `dict`

When the plant is **found**, return:
```python
{"found": True, "plant": <the full plant dict from _plant_db>}
```

When the plant is **not found**, return:
```python
{"found": False, "name": <normalized input>, "message": <helpful string>}
```

---

### Design Decisions

*Complete the two blank fields below before writing code. The others are pre-filled for you.*

---

#### Input normalization

Strip leading/trailing whitespace and convert to lowercase before any comparison.

```python
normalized = plant_name.strip().lower()
```

---

#### Search order

Search in this order: direct key → display name → aliases. Keys are the fastest
lookup (O(1) dict access), so check those first. Display names are the next most
likely match for clean user input. Aliases are the broadest net, so they go last.

```
1. Direct key match: normalized in _plant_db
2. Display name match: plant["display_name"].lower() == normalized
3. Alias match: normalized in [alias.lower() for alias in plant["aliases"]]
```

---

#### Alias matching approach

*Aliases are stored as a list of strings. How will you check if the normalized input matches any alias in the list? Write your approach in pseudocode or plain English.*

```
Normalize the query to lowercase. For each plant, build a lowercase list of all its
aliases and check if the query is in that list:

  query in [a.lower() for a in plant.get("aliases", [])]

This handles plants with no aliases (empty list) gracefully via .get() default.
```

---

#### Not-found message

*When a plant isn't found, the agent will read your message and use it to decide what to tell the user. Write the exact string you'll return — make it useful to the agent, not just to a human reading logs.*

```
"'{plant_name}' is not in the plant database. Do not invent specific care instructions
for this plant. Acknowledge it is not in your database, then offer general guidance
based on the plant's type or family if identifiable. Plants available in the database:
Pothos, Snake Plant, ZZ Plant, ..."

The message embeds a direct instruction to the LLM alongside the factual gap, so the
agent knows both what data is missing and how to respond.
```

---

#### Implementation Notes

*Fill this in after implementing and running the app.*

**Test: does `"devil's ivy"` return the pothos entry?**
```
yes
```

**Test: does `"SNAKE PLANT"` return the snake plant entry?**
```
yes
```

**One edge case you discovered while implementing:**
```
get_seasonal_conditions() receives tool_call.function.arguments from the LLM, which
can be an empty string or the JSON string "null" when no season is provided. Both
json.loads("") and json.loads("null") fail or return None, not {}. Fixed by:
  json.loads(tool_call.function.arguments or "{}") or {}
```

---

## Function 2: `get_seasonal_conditions()`

### Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `season` | `str \| None` | One of `"spring"`, `"summer"`, `"fall"`, `"winter"`, or `None` to auto-detect |

**Output:** `dict`

The full season dict from `_season_data`, plus one additional field:

| Added field | Type | Value |
|-------------|------|-------|
| `"detected_season"` | `bool` | `True` if auto-detected from the month; `False` if season was passed as an argument |

---

### Design Decisions

*This function is pre-implemented — read through these fields and the code before working on `lookup_plant`.*

---

#### Auto-detection logic

When `season` is `None`, get the current calendar month with `datetime.now().month`
and look it up in the `_MONTH_TO_SEASON` dict, which maps month numbers to season strings.

```python
current_month = datetime.now().month
season_key = _MONTH_TO_SEASON[current_month]
```

---

#### Season validation

If the caller passes an invalid season string (e.g., `"monsoon"`), the function
falls back to auto-detection — same as if `None` were passed. The `VALID_SEASONS`
set acts as the gate:

```python
VALID_SEASONS = {"spring", "summer", "fall", "winter"}
if season and season.lower() in VALID_SEASONS:
    ...  # use provided season
else:
    ...  # auto-detect
```

---

#### Return structure

The full season dict from `_season_data`, plus a `detected_season` boolean. Example for spring:

```python
{
    "season": "spring",
    "watering": "Increase watering frequency as plants break dormancy ...",
    "fertilizing": "Resume feeding with a balanced fertilizer ...",
    "light": "Days are lengthening — move plants closer to windows ...",
    "pests": "Watch for spider mites and aphids as temperatures rise ...",
    "detected_season": True   # True = auto-detected; False = caller specified
}
```

---

#### Implementation Notes

*Fill this in after testing.*

**Test: does calling with `season=None` return the correct season for the current month?**
```
Current month: 6
Expected season: summer
Returned season: Summer
```

**Test: does calling with `season="winter"` return winter data regardless of the current month?**
```
yes
```
