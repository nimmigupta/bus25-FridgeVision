Here’s a **ready-to-paste prompt for OpenAI Codex** (to generate Python code) that implements your CEO’s holiday push app. It tells Codex exactly what to build, which libraries to use, how the UI should behave, and how to wire in **Gemini 2.5 Pro** (vision) and **Gemini Flash** (recipes) via **Google AI Studio**—plus favorites, calorie targets, nutrition notes, and robust error handling.

---

# Prompt for OpenAI Codex

**Role**: You are a senior Python engineer. Generate production-quality code, comments, and a runnable app with clear structure.

**Project**: “FridgeVision Chef” — A simple, user-friendly Python web app for a refrigerator company to boost Christmas sales by showcasing AI features: recognize items in a fridge photo and generate tailored recipes.

## Tech Stack (Python only)

* **Framework**: FastAPI (backend) + Jinja2 templates (simple, clean HTML/CSS/vanilla JS front-end).
* **Image inputs**: file upload, webcam capture, and clipboard paste (JS paste handler).
* **Model integrations (Google AI Studio)**:

  * **Gemini 2.5 Pro (Vision)** for food item recognition.
  * **Gemini Flash** for recipe generation.
* **Google AI SDK**: `google-generativeai` (a.k.a. Google AI Python SDK).
* **Storage**:

  * **SQLite** for favorites (recipes users save).
  * **LocalStorage** (browser) for lightweight user preferences (veg/non-veg, cuisines, health goals).
* **Env config**: `.env` with `GOOGLE_API_KEY`. Settings page writes/loads this at runtime safely on server side.

> Do **not** require an OpenAI API key for runtime; OpenAI Codex is only the code generator here.

## Features & Requirements

1. **Home / Recognize**

   * UI allows **three** ways to provide an image:

     * Upload (drag-drop or select).
     * **Take Photo** (webcam).
     * **Paste** an image (Ctrl/Cmd+V) into a visible drop zone.
   * When user submits an image:

     * Call **Gemini 2.5 Pro (Vision)** to identify visible **food items**, quantities (rough), and confidence.
     * If **not** a food image (or low confidence), show a friendly, clear message:
       “This doesn’t look like food. Please try another image.”
       Provide retry controls and tips.
     * If valid: display a **tag list** of identified items (e.g., “eggs”, “spinach”, “milk”).

2. **Recipe Generator**

   * Show a compact form with:

     * Dietary preference: **Healthy**, **Vegetarian**, **Non-Vegetarian** (multi-select: allow combinations where logical).
     * Cuisine type (chips with common options + free-text).
     * **Calorie target** input (optional): e.g., “Create a ~600 kcal recipe.”
   * When user clicks **Generate Recipes**:

     * Send **(items + preferences + calorie target)** to **Gemini Flash**.
     * Return **at least 5 distinct recipes**. Each recipe must include:

       * Title, 1-paragraph description,
       * Ingredients (with approximate quantities),
       * Step-by-step instructions (5–10 steps),
       * Estimated calories per serving and **macros** (protein/fat/carbs) approximate,
       * **Nutrition angle**: short note on why it’s healthy (e.g., “high-protein, helps weight management”),
       * Cuisine tag and dietary tags.
     * Display results in clean cards with a **“Save to Favorites”** button.

3. **Favorites**

   * Store saved recipes in **SQLite** with fields: id, title, items_used (JSON), tags, calories, macros, full_recipe_json, created_at.
   * A **Favorites** page shows saved items with search/filter by dietary tag/cuisine and a **“Remove”** button.
   * Export a recipe to **printable view** (simple CSS print styles).

4. **Healthy Eating Section**

   * A **“Healthy Eating (High-Protein)”** page:

     * Static + dynamic content:

       * Short educational blurbs about high-protein food choices.
       * When items detected, suggest **protein-forward swaps** (e.g., Greek yogurt for sour cream).
     * Provide 3 quick-start recipe ideas (Gemini Flash mini-prompts) emphasizing **high protein**.

5. **Settings**

   * Page to configure:

     * **Google AI Studio API key** (GOOGLE_API_KEY).
     * Model selection dropdowns (default: Gemini 2.5 Pro for vision; Gemini Flash for recipes).
     * Safety settings (temperature, max tokens) with sensible defaults.
   * Validate and persist to server env (or a secure server-side JSON file). Never expose the API key in client logs.

6. **Error Handling & UX**

   * Graceful failures with user-friendly messages and a **retry** button.
   * Loading spinners; disable buttons while awaiting responses; timeouts with helpful tips.
   * Validate calorie target (integer range).
   * If models return fewer than 5 recipes, retry once; otherwise display as many as available with a notice.

7. **Sales Tie-In (subtle)**

   * On recipe cards, include a small **“Make it easier with our new fridge features”** link leading to a **promo banner** (mock placeholder route) highlighting smart-inventory tracking—positioned tastefully (non-intrusive).

## Project Structure

```
fridgevision_chef/
  app.py                 # FastAPI main
  routers/
    __init__.py
    pages.py             # page routes (home, recipes, favorites, healthy, settings)
    api.py               # API endpoints (recognize, generate, favorites CRUD)
  services/
    gemini.py            # wrappers for Gemini 2.5 Pro (vision) and Gemini Flash (text)
    nutrition.py         # simple helpers for calorie/macros estimation & notes
    storage.py           # SQLite helpers (favorites)
    settings.py          # load/save API settings
  templates/
    base.html
    home.html
    recipes.html
    favorites.html
    healthy.html
    settings.html
    printable.html
  static/
    css/styles.css
    js/paste.js          # handles clipboard image paste
    js/webcam.js         # capture image from webcam
    js/common.js
  models/
    recipe.py            # Pydantic models
  schema.sql             # SQLite schema (favorites)
  .env.example
  requirements.txt
  README.md
```

## Key Implementation Details

### 1) Install

`requirements.txt` should include:

```
fastapi
uvicorn[standard]
jinja2
python-multipart
pydantic
pydantic-settings
python-dotenv
google-generativeai
Pillow
sqlite-utils
```

### 2) Gemini Service (`services/gemini.py`)

* Initialize Google AI with `genai.configure(api_key=...)`.
* **Vision (Gemini 2.5 Pro)**:

  * Function: `identify_food_items(image_bytes) -> dict` returning:

    ```python
    {
      "is_food": bool,
      "items": [{"name": "spinach", "confidence": 0.92}, ...],
      "notes": "Detected leafy greens and dairy."
    }
    ```
  * Prompt the model to **only** return items if clearly edible; else set `is_food=False`.
* **Recipes (Gemini Flash)**:

  * Function: `generate_recipes(items, dietary_prefs, cuisine_tags, calorie_target) -> list[Recipe]`
  * Ensure **≥5** recipes. Include calories & macros estimates, plus a short “why healthy” note.

### 3) Nutrition Helpers (`services/nutrition.py`)

* Simple reference dict for macros per 100g for common items (approx).
* Functions to:

  * Estimate calories/macros per recipe from ingredient list.
  * Provide **weight-loss notes** when calories are moderate and protein is high.

### 4) Favorites (`services/storage.py`)

* SQLite with table `favorites(id INTEGER PK, title TEXT, tags TEXT, calories INT, macros TEXT, recipe_json TEXT, created_at TEXT)`.
* CRUD functions `save_favorite`, `list_favorites`, `delete_favorite`.

### 5) Settings (`services/settings.py`)

* Load/save API settings to a **server-side** JSON file (e.g., `settings.json`) and environment variables via `.env`.
* Validate key presence before calls. Return errors to UI if missing.

### 6) Routes

* `GET /` → `home.html` (upload, webcam, paste UI).
* `POST /api/recognize` → run vision, return items or not-food message.
* `POST /api/generate` → run recipes; return list.
* `POST /api/favorites` and `DELETE /api/favorites/{id}`.
* `GET /favorites`, `GET /healthy`, `GET /settings`, `POST /settings`.

### 7) Front-End (Jinja + vanilla JS)

* **Home**: Drop zone with paste support (`paste.js` listens for `paste` events, reads `ClipboardItem` images).
* **Webcam**: Use `getUserMedia` to capture a frame to canvas → blob → send.
* **Forms**: dietary chips (Healthy, Vegetarian, Non-Vegetarian), cuisine chips & free-text, calorie input.
* **Toasts** for success/error. **Spinner** while awaiting API.
* **Favorites**: cards with remove+print.
* **Healthy**: curated content + dynamic suggestions from detected items.

### 8) Prompts to Gemini (examples)

* **Vision** (system style):

  ```
  You are a food-vision assistant. Determine if an image prominently contains edible food.
  If not food, reply: {"is_food": false, "items": [], "notes": "<short reason>"}.
  If food, reply JSON like:
  {
    "is_food": true,
    "items": [{"name":"tomato","confidence":0.91}, ...],
    "notes":"<short note>"
  }
  Only return JSON.
  ```
* **Recipes** (Gemini Flash):

  ```
  You are a culinary planner. Given these items: <list>,
  dietary prefs: <prefs>, cuisine tags: <tags>, calorie target: <kcal or none>.
  Produce at least 5 distinct recipes. JSON array of objects:
  {
    "title": "...",
    "description": "...",
    "ingredients": [{"item":"...", "qty":"..."}],
    "steps": ["...", "..."],
    "calories_per_serving": 0,
    "macros": {"protein_g": 0, "fat_g": 0, "carbs_g": 0},
    "why_healthy": "...",
    "tags": ["cuisine:...","veg|non-veg","high-protein" | "balanced"]
  }
  Ensure concise, practical ingredient lists using provided items when possible.
  ```

### 9) Validation & Edge Cases

* Reject images > 10MB with a clear message.
* If **no API key** configured, block calls and route user to **Settings** with guidance.
* If <5 recipes returned, attempt one retry with slightly higher temperature; otherwise show what’s available + note.
* When confidence < 0.6 overall, treat as “not food” with guidance.

### 10) Run & Docs

* `README.md` must include:

  * Setup: create `.env` with `GOOGLE_API_KEY=...`
  * Install & run: `pip install -r requirements.txt` then `uvicorn app:app --reload`.
  * Browser usage steps (upload, webcam, paste; generating recipes; saving favorites).
  * Notes on privacy (keys stored server-side; preferences in localStorage).

### 11) Acceptance Tests (lightweight)

* Unit tests for `identify_food_items` mock response parsing.
* Unit tests for recipe schema validation (≥5, fields present).
* Integration test: POST image → items → POST generate → 5 recipes.
* Favorites CRUD tests.

## Deliverables

* Complete codebase with all files, comments, and minimal CSS for a clean, friendly UI.
* Sensible defaults; no placeholder TODOs left.

---

**Output**: Provide the full Python project with files laid out as above, plus the HTML/CSS/JS templates and a `README.md`.
