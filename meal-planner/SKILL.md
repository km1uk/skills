---
name: meal-planner
description: Generates a balanced 7-day dinner/main course meal plan from the user's Todoist recipe list. Use this skill whenever the user asks to plan meals, generate a weekly meal plan, pick recipes for the week, or says anything like "what should I cook this week". Also use it when the user wants to update, refresh, or adjust an existing meal plan. The skill handles fetching recipes from Todoist, applying constraints (protein variety, time, priority), and tagging the chosen recipes with the appropriate priority level.
---

# Meal Planner Skill

Generates a balanced 7-day main course meal plan drawn from the user's Todoist recipe list, then tags the chosen recipes in Todoist with the next available priority level.

## Step 1 – Find the Recipe project

Search Todoist for "recipe" to locate the Recipe ideas project and get its ID.

```
search("recipe") → find project with title like "Recipe ideas"
```

## Step 2 – Fetch candidate recipes

Pull two pools in parallel:

1. **Active tasks** from the Recipe ideas project (all of them, limit 100+)
2. **Completed tasks** from the same project — scan recent windows (e.g. last 6 months in 3-month chunks) to surface tried-and-tested meals

Note the completion date of any completed tasks — this is used in Step 4 to favour fresher recipes.

## Step 3 – Identify the next available priority tier

Todoist uses p1–p4 (p1 = highest, p4 = lowest/default). The recipe list uses priority as a "cooking queue":
- p4 = backlog (not yet planned)
- p3, p2, p1 = progressively more imminent

**Before suggesting recipes**, check what priority levels already have tasks assigned in the project. The new meal plan should use the **next free tier** (e.g. if p4 and p3 are taken, use p2). Explain this logic to the user briefly.

## Step 4 – Apply constraints and build the plan

Default constraints (apply unless user overrides):
- **7 days**, one main course per day
- **No repeated primary protein** across the 7 days (e.g. no two chicken dishes). Proteins to track: chicken, beef, pork, lamb, fish/seafood, duck, game, vegetarian/vegan
- **Priority filter**: **only pick from p4 tasks**. p1, p2, and p3 all mean the recipe is already planned or in progress — never select them. This is a hard rule, not a default.
- **Recency filter**: deprioritise recipes completed within the last ~2 months to avoid repetition. If the active list is thin, these can be included but flag them as recently cooked.
- **Completed recipes** can serve as inspiration / fallbacks if the active list is thin

**Accompaniment awareness**: Many recipes implicitly require a carb or side (e.g. curry → rice or bread, stew → potatoes or crusty bread, stir-fry → noodles or rice). When selecting the 7 recipes, mentally tag the implied accompaniment for each and ensure the week isn't monotonous — e.g. avoid a week where every meal lands on rice, or where 4+ dishes need bread. Aim for variety across: rice, noodles, bread/flatbread, potatoes, pasta, and grain-free/standalone dishes. Surface the accompaniment alongside each recipe in the plan (Step 5).

**Common constraint the user may add:**
- "No time-consuming recipes" / "weeknight friendly" → exclude anything that reads as a slow-cook, braise, or long-prep dish (e.g. cheeks, short ribs, confit, slow-roast)
- "Even at the weekend" → applies the quick-cook filter to all 7 days
- Ingredient availability → swap out hard-to-source items (specialist fish, offal cuts) for accessible alternatives from the list

When the user asks to swap a recipe, always pick a replacement from the same project list, respecting the protein-variety rule and accompaniment balance.

## Step 5 – Look up recipe details (when needed)

If the user or the plan requires more detail about a specific recipe (e.g. to check prep time, identify the protein, confirm an accompaniment, or help with cooking):

1. **Check the Todoist task first** — fetch the task's description, comments, and any photo attachments. The full recipe is often stored there.
2. Only if the task contains no useful detail should you fall back to searching online for a close match.

### Retrieving a photo/file attached to a Todoist comment

Recipe tasks often have a photo of a physical recipe page (magazine, cookbook) attached as a comment rather than typed out. To read one:

1. Call `find-comments` with the task's `taskId`.
2. If a comment has a `fileAttachment` object, grab the `fileUrl` field from it (this is a `files.todoist.com` URL — it is **not** directly fetchable with a generic web-fetch tool, since it wasn't returned by a search and isn't a URL the user pasted).
3. Call **`view-attachment`** with that `fileUrl`. This is the correct, dedicated tool for this job — it returns the image inline (or text/PDF content as appropriate) regardless of which other web/file tools are or aren't available in the session.
4. **This tool is deferred** — if it doesn't appear in your available tools yet, run `tool_search` with a query like "Todoist view attachment file" to load it before calling it. Don't assume the attachment is unreadable just because `view-attachment` isn't listed yet.
5. Do not attempt to read `files.todoist.com` URLs via a generic bash `curl`/`wget` or a general-purpose web-fetch tool — these will fail (bash's network allowlist won't include that domain, and web-fetch tools typically require the URL to have come from a prior search or from the user directly). `view-attachment` is the one reliable path.
6. Once viewed, transcribe the ingredients/method from the image into the task's context for that planning session (e.g. quote it back when building the shopping list) rather than re-fetching the same image repeatedly.

### When a recipe can't be found or confirmed

Sometimes none of the above works: the task has no useful description/comments/attachment, and an online search doesn't turn up the specific recipe (e.g. it's a page from a print magazine or a personal cookbook with no online presence). **Don't silently guess ingredients and present them as fact.** Instead, stop and ask the user how they'd like to proceed, using an interactive prompt (e.g. `ask_user_input_v0`) with these options:

- **Use a recipe found online** — proceed with the closest online match Claude found during search, clearly labelled as a substitute rather than the exact source
- **Add a more specific URL** — the user provides a better/direct link to the actual recipe, which Claude then fetches
- **Attach a file in chat** — the user uploads a photo or file of the recipe directly in the conversation (fastest path when the recipe is a magazine/cookbook page); do not confuse this with a Todoist attachment lookup — this is a fresh upload the user sends
- **Skip it** — exclude the recipe's ingredients from the shopping list entirely

Ask this **per recipe**, not once for the whole plan, since different recipes may hit this issue for different reasons.

If the user picks **Skip**, don't just omit the recipe quietly — surface it in the shopping list output as a visible warning (e.g. a flagged line like "⚠️ Sea bream with courgettes & chorizo — ingredients not included, recipe not found") so the user notices the gap before they shop rather than discovering a missing dish at cooking time.

If the user picks **"Use a recipe found online"** and confirms a specific link, save that link back to the Todoist task so it doesn't need re-finding next time:
- Update the task's `content` to include the URL (Todoist tasks commonly use markdown-style links, e.g. `[Recipe name](url)`), or add it to the task's `description` if the existing content/title should stay as-is — prefer whichever keeps the task readable and consistent with how other recipe tasks in the project are formatted.
- Do this via `update-tasks` (or the equivalent task-update tool) right after the user confirms the link — don't wait until the end of the session, in case the plan changes further.
- Briefly confirm to the user that the task has been updated with the link.

## Step 6 – Present the plan

Display as a visual card or table showing:
- Day of week
- Recipe name (with source/page ref if present)
- Primary protein tag
- Suggested accompaniment (e.g. "serve with rice", "needs flatbread")
- Any relevant labels (weekend, fire, curry, etc.)

## Step 7 – Handle corrections interactively

The user may:
- Reject a recipe (hard to find, too slow, wrong priority level) → swap it out, explain reasoning
- Point out a priority mismatch (e.g. "that's p3 not p4") → correct and re-check the full plan
- Ask to relax or tighten constraints

Always re-verify constraints after each swap before confirming the plan.

## Step 8 – Tag chosen recipes in Todoist

Once the plan is confirmed:

1. Identify the **target priority** (next free tier — see Step 3)
2. For each of the 7 chosen recipes:
   - If it's an **active task**: update priority using `update-tasks`
   - If it's a **completed task**: first uncomplete it with `uncomplete-tasks`, then update priority
3. Confirm all 7 updates to the user

```
update-tasks([{id, priority: "p2"}])   ← or whichever tier is next free
```

## Notes

- Todoist priority strings: "p1", "p2", "p3", "p4" — never integers
- The Recipe ideas project ID can change per user — always look it up fresh via search
- Completed tasks in Todoist are not returned by `find-tasks`; use `find-completed-tasks` with `projectId` and date windows
- When scanning completed tasks, use 3-month windows (API limit) and go back as far as needed
- Labels on recipe tasks (e.g. "weekend", "game", "curry", "fire") are useful metadata — surface them in the plan display
