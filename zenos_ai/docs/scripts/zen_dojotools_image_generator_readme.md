# Zen DojoTools Image Generator — v5.1.0

**File:** `packages/zenos_ai/dojotools/dojotools_image_generator.yaml`
**Script:** `zen_dojotools_generate_image`
**MCP-exposed**

Generates an image via `ai_task.generate_image` (DALL·E-class model) and writes it to one of six named display slots, each backed by a trigger-based `image.*` entity.

---

## Requirements

Only `ai_task.openai_ai_task_3` (`gpt-image-1.5`) supports `GENERATE_IMAGE` in this install. Set `input_text.zenos_image_task` to that entity_id after any reload — the script falls back to it when no `entity_id` is passed explicitly.

---

## Slot Architecture

Each slot is a trigger-based `image.*` entity (the canonical `ai_task` pattern, not a template sensor):

| Slot | Entity | Intended use |
|------|--------|--------------|
| `canvas` (default) | `image.zen_image_canvas` | Main display |
| `portrait` | `image.zen_image_portrait` | AI portrait |
| `wallpaper` | `image.zen_image_wallpaper` | Wallpaper / background |
| `id_pic` | `image.zen_image_id_pic` | Identity / selfie |
| `dashboard_snap` | `image.zen_image_dashboard_snap` | Dashboard snapshot |
| `home_state` | `image.zen_image_home_state` | Home state visualization |

Flow: the script calls `ai_task.generate_image`, then fires `zen_event` (`kind: image_generated`, filtered by `slot`). The matching trigger entity fetches the URL once (auth signature valid ~1h) and HA's image proxy caches it permanently — the signed URL itself doesn't need to survive.

**Restart persistence:** after fetch, `image.snapshot` writes the fetched image to disk with a two-path strategy:
- Primary: `/config/www/zen_images/<slot>.jpg` — `continue_on_error: true`, silently no-ops if the `zen_images/` folder doesn't exist yet.
- Fallback: `/config/www/zen_<slot>.jpg` (flat `www/`) — always fires regardless.

Create `/config/www/zen_images/` once to activate the organized path; until then every generation still lands via the flat fallback.

---

## Fields

| Field | Required | Description |
|-------|----------|-------------|
| `image_prompt` | Yes | Descriptive prompt — be specific about subject, style, lighting, composition. |
| `slot` | No | One of the six slots above. Default `canvas`. |
| `entity_id` | No | Specific `ai_task.*` entity to use. Defaults to `input_text.zenos_image_task`, then HA's preferred entity. |
| `mode` | No | Pass `tool_manifest` to get the tool's self-description. Not a dispatch mode otherwise — this script has exactly one real operation (generate). |
| `caller_token` | No | Opaque pass-through for caller correlation. |

---

## Response Shape

```yaml
status: success | error
message: "Image written to slot: canvas." | "No url returned from ai_task."
result:
  slot: canvas
  url: "..."                          # signed URL from ai_task (short-lived)
  media_source_id: "..."
  revised_prompt: "..."               # if the model revised your prompt
  model: "..."
  width: 1024
  height: 1024
  prompt: "..."                       # your original prompt
  snapshot_fallback: "/config/www/zen_canvas.jpg"
  snapshot_organized: "/config/www/zen_images/canvas.jpg"
  storage_hint: "Saved to zen_images/canvas.jpg and fallback www/zen_canvas.jpg."
  setup_tip: "Create /config/www/zen_images/ to activate organized slot storage. Fallback www/zen_<slot>.jpg always fires."
caller_token: "..."
```

`status: error` with an empty `url` means `ai_task.generate_image` didn't return one — check the configured `ai_task` entity is actually `gpt-image-1.5`-capable.

---

## Example

```yaml
image_prompt: "A cozy reading nook by a window, warm afternoon light, watercolor style"
slot: wallpaper
```

---

## Notes

- The 3-second `delay` between firing `zen_event` and taking the snapshot gives the trigger entity time to fetch the URL before `image.snapshot` runs. If a generation is consistently landing on the fallback-only path with no organized copy, that's expected until `/config/www/zen_images/` exists — not a bug.
- `entity_id` filters strictly to the `ai_task` domain — passing a non-`ai_task` entity is rejected by the selector before the script ever runs.
