# Zen DojoTools Print Shop — v0.1.0-experimental

*IPP-based printer control: submit text print jobs and trigger maintenance on household printers*

---

> **EXPERIMENTAL.** This is a v0.1 print path built on a HACS integration and a manual-configuration workflow. It has known gaps (no PDF/binary printing, best-guess maintenance URLs, hardcoded ink-sensor naming). Treat it as "may work for your printer," not a supported feature. Per the file's own header: if `test_page` doesn't produce real output on your hardware, this path is considered abandoned.

---

## Overview

Print Shop submits plain-text print jobs and triggers basic maintenance (head cleaning, nozzle check) on household printers, using Home Assistant's native IPP integration for job submission plus the Rudd-O `homeassistant-ipp-printing` HACS bridge to actually get bytes to the printer. Maintenance actions bypass IPP entirely and hit the printer's embedded HTTP interface directly.

**Entity:** `script.zen_dojotools_print_shop`

---

## Prerequisites (manual setup — do this before using the tool)

1. **Install HA's native IPP integration** (Settings → Devices & Services → Add Integration → IPP). Point it at the printer directly (e.g. `192.168.1.120:631/ipp/print`) or at a CUPS queue. This creates a status sensor, e.g. `sensor.epson_wf_7840_series` — note the entity_id, you'll need it. Repeat once per household printer.
2. **Install Rudd-O's `homeassistant-ipp-printing` via HACS.** This is a single confirm-only config step — printer details are not configured inside Rudd-O itself, only in the IPP integration and this tool's `configure` mode.
3. **Expose the IPP sensor via Settings → Voice Assistants** (needed for the entity to be usable by the automation layer).
4. **Run `mode=configure`** with `epson_ip` and `epson_entity` to seed the `print_shop` drawer in the AI user cabinet.

Without all four steps, most modes will either no-op or fail against an unconfigured/unreachable printer.

---

## Modes

| Mode | Description |
|---|---|
| `configure` | Write/update the `print_shop` cabinet drawer: `epson_ip`, `epson_entity`, `paper_size`, `orientation` |
| `test_page` | Render and print a timestamped test page (printer name, state, time) |
| `print_text` | Render and print the `content` field as plain text |
| `print_image` | Print a generated image from a named slot (`canvas`, `portrait`, `wallpaper`, `id_pic`, `dashboard_snap`, `home_state`) — reads from `/config/www/zen_images/<slot>.jpg` via the local HA `/local/` path |
| `status` | Printer state + ink levels. Reads HA entity state only — no IPP call |
| `list_jobs` | All IPP jobs (`job_filter`: `all` \| `incomplete` \| `complete`) |
| `clean` | Trigger a head-cleaning cycle via the printer's HTTP maintenance UI (~3 minutes) |
| `nozzle_check` | Trigger a nozzle-check pattern via the printer's HTTP maintenance UI |
| `help` | Field reference + examples |
| `tool_manifest` | Standard manifest response |

**Default mode is `status`** (per the `_mode` fallback), not `help` — check the printer before assuming nothing happens on a bare call.

---

## Key Fields

| Field | Modes | Description |
|---|---|---|
| `epson_ip` | `configure` | LAN IP of the printer |
| `epson_entity` | `configure` | entity_id of the IPP status sensor |
| `content` | `print_text` | Text to print |
| `slot` | `print_image` | Which generated image slot to print |
| `job_filter` | `list_jobs` | `all` \| `incomplete` \| `complete`, default `all` |
| `printer_entity` | any | Overrides the configured default IPP entity — for multi-printer setups |
| `paper_size` | `configure` | IPP media name, default `na_letter_8.5x11in` |
| `orientation` | most modes | `portrait` \| `landscape` \| `reverse-portrait` \| `reverse-landscape`, default `portrait` |

`printer_entity` defaults to the `print_shop` cabinet's `epson_entity` value if not passed explicitly.

---

## How Printing Actually Works

`test_page`, `print_text`, and `print_image` all go through the same two-step path:

1. `simpleimageraster.draw` renders the content (text or a fetched image) to a JPEG in memory.
2. `ipp_printing.print` (the Rudd-O bridge) sends that JPEG to the printer via IPP.

There is no direct text-to-printer path — everything is rasterized to an image first. This is also why PDF/binary printing isn't supported: base64 file encoding of arbitrary binary content isn't feasible from a Jinja2 template context, so only content this script can itself render (plain text, or an image already on disk under `/config/www/zen_images/`) can be printed.

`clean` and `nozzle_check` are different: they POST directly to hardcoded printer-firmware HTTP paths (`/PRESENTATION/HTML/TOP/CLEANING.HTML`, `.../NOZZLECHECK.HTML`) using `epson_ip` from the cabinet — no IPP, no Rudd-O involvement.

---

## Known Limitations (v0.1)

- **No PDF/binary printing.** Only text (rendered as an image) and pre-existing local images under `/config/www/zen_images/<slot>.jpg`.
- **Maintenance URLs are best-guess Epson WF-7840 paths.** `clean`/`nozzle_check` were written against that specific model's embedded web server. Validate on first run against your hardware and expect to adjust the URLs for a different printer.
- **Ink-level sensors are hardcoded to the WF-7840 sibling-entity naming pattern** (`<entity>_black_ink`, `_cyan_ink`, etc). `mode=status` will report `unknown` for any printer whose IPP integration doesn't create ink sensors in that exact naming shape.
- Single-printer defaults are baked into the cabinet drawer; use `printer_entity` per-call for anything beyond the configured default.

---

## Examples

```yaml
# One-time setup
zen_dojotools_print_shop:
  mode: configure
  epson_ip: 192.168.1.120
  epson_entity: sensor.epson_wf_7840_series

# Check status
zen_dojotools_print_shop:
  mode: status

# Print a test page
zen_dojotools_print_shop:
  mode: test_page

# Print text
zen_dojotools_print_shop:
  mode: print_text
  content: "Hello from ZenOS"

# List pending jobs
zen_dojotools_print_shop:
  mode: list_jobs
  job_filter: incomplete

# Trigger head cleaning
zen_dojotools_print_shop:
  mode: clean
```

---

## Dependencies

| Dependency | Purpose |
|---|---|
| HA native IPP integration | Printer status sensor + `ipp_printing` job submission transport |
| Rudd-O `homeassistant-ipp-printing` (HACS) | Provides the `ipp_printing.print` service |
| `simpleimageraster.draw` (HACS/custom component) | Text/image → JPEG rasterization |
| `script.zen_dojotools_filecabinet` | `print_shop` drawer read/write on the AI user cabinet |
| `rest_command.epson_print_clean` / `epson_print_nozzle` | Direct printer HTTP maintenance calls |

---

## Version History

| Version | Change |
|---|---|
| 0.1.0-experimental | Initial release. Text + image printing via IPP/Rudd-O, WF-7840-specific maintenance triggers. |
