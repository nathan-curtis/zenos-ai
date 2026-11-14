# Zen DojoTools Print Shop — v0.6.4-experimental

*IPP-based printer control: submit text print jobs and trigger maintenance on household printers*

---

> **EXPERIMENTAL.** This is a v0.1 print path built on a HACS integration and a manual-configuration workflow. It has known gaps (no PDF/binary printing, best-guess maintenance URLs, hardcoded ink-sensor naming). Treat it as "may work for your printer," not a supported feature. Per the file's own header: if `test_page` doesn't produce real output on your hardware, this path is considered abandoned.

---

> **Response envelope (2026.10.0):** `zen_dojotools_print_shop` returns the standard OS envelope — `{status, mode, tool, result, system_message, caller_token}` (see [`envelope()`](../custom_templates/zen_os1_jinja.md#envelopestatus-mode-result-tool-caller_token--canonical-response-shape)). The response fields documented on this page live under `result`; top-level `status` is the generic `success`/`error` execution status. Read `.result` when consuming a response via `response_variable`.

> **Backstage pass required (2026.11.0).** An agent without `zenos.agent.basic` gets this tool's `help`, `tool_manifest` and `kfc_manifest` only. Anything else answers `status: error`, `reason: "guarded: onboarding_required: ..."` and `next_step: {tool: zen_flynn_oobe, mode: run}`, and writes one `zen_entry_gate` warning to the system log. Calls no user started (boot, Flynn, automations, the scheduler) pass. See [the onboarding gate](../developer_standards.md#the-onboarding-gate).

## Overview

Print Shop submits plain-text print jobs and triggers basic maintenance (head cleaning, nozzle check) on household printers, using Home Assistant's native IPP integration for job submission plus the Rudd-O `homeassistant-ipp-printing` HACS bridge to actually get bytes to the printer. Maintenance actions bypass IPP entirely and hit the printer's embedded HTTP interface directly.

**Entity:** `script.zen_dojotools_print_shop`

---

## Prerequisites (manual setup — do this before using the tool)

1. **Install HA's native IPP integration** (Settings → Devices & Services → Add Integration → IPP). Point it at the printer directly (e.g. `192.0.2.10:631/ipp/print`) or at a CUPS queue. This creates a status sensor, e.g. `sensor.epson_wf_7840_series` — note the entity_id, you'll need it. Repeat once per household printer.
2. **Install Rudd-O's `homeassistant-ipp-printing` via HACS.** This is a single confirm-only config step — printer details are not configured inside Rudd-O itself, only in the IPP integration and this tool's `configure` mode.
3. **Expose the IPP sensor via Settings → Voice Assistants** (needed for the entity to be usable by the automation layer).
4. **Run `mode=configure`** with `epson_ip` and `epson_entity` to seed the `print_shop` drawer in the AI user cabinet. If `epson_entity` isn't configured, the tool uses the first `sensor.*` carrying the `printer` label.

Without all four steps, most modes will either no-op or fail against an unconfigured/unreachable printer.

**Optional, Epson printers:** the `epson_workforce` HACS integration (`ha-epson-workforce`, sensor-only) adds maintenance-box level, page counters and scanner status. `status` matches it to the same printer by device model, so no entity IDs are configured. It has no print path and no job state, so the IPP integration stays the print target. Without it, `status` still answers from IPP alone (`sources.maintenance: not available`).

**3D printers (read-only):** a Bambu Lab printer from the `ha-bambulab` integration is listed by `printers` as `fdm_bambu`. Pass its "Print status" sensor as `printer_entity` and `status` returns a digest built from the printer's own entities, found by device and entity name: state, stage, progress, remaining minutes, layers, temperatures, job, nozzle, AMS trays (each with an `unknown` flag) and humidity, alarms, door, SD card, firmware, connection, control and camera. **Nothing sends a command to a 3D printer.** Control is reported and fails closed: `control.available` is true only when MQTT encryption is off or Developer LAN mode is on, hybrid control is not blocked, and all three sensors are known. ZenOS works with whatever mode you have enabled and never asks you to change a printer's setup. OctoPrint printers are listed only; their status is not built yet.

**The `print_shop` component (KF5).** Print Shop hosts its own Kung Fu component, seeded by `mode=summary`, so the house keeps a Kata on its printers.
- **Certificate:** `zenos.media.print_status` (level 1, `st_ok: isl`). `summary` needs it; every other read mode stays open. The seed is tagged `summary@zenos.media.print_status:1`, and the component's `meta.requires` names the same cert, so it runs, and shows in the prompt, only for an agent that holds it. Grant it with `cert_stages=isl`; on an install with live acks, each stage asks on a phone.
- **Live drawer:** the cert names `summary` in its `modes` and allows stage `l`, so a live drawer mounted on `summary` reads without being on FileCabinet's callout whitelist (FileCabinet 6.38.0).
- **Triggers:** `ha_start`, `daily_midnight` and `force_summary`, plus the Scheduler's `print_done` (a Bambu Lab print finished or failed) while `job_end_summary` is on. Run `mode=configure`, then `bootstrap_kfc`, to apply a change.
- **Escalation:** each printer is scored (failed print or alarm 6, paused or finished job 4, paper or ink 3, running 1). `suggested_act_event` is Postman, at urgency 4 and up, only when `notify_on_job_end` is on. A 3D printer in `finish` counts as done for 12 hours.

**Job-end settings,** stored in the `print_shop` drawer by `mode=configure`, all **off by default**: `job_end_camera_look` (one vision call), `job_end_summary` (one summarizer run) and `notify_on_job_end` (a Postman push). `status` shows them under `job_end_steps`, and `brief` under `steps`. Read modes never run an inference step.

**Labels:** put `printer` only on each printer's IPP sensors (status, ink, uptime). The default printer is the first `printer`-labeled sensor that has an IPP address, so an ink sensor carrying the label is never picked.

---

## Modes

| Mode | Description |
|---|---|
| `configure` | Write/update the `print_shop` cabinet drawer: `epson_ip`, `epson_entity`, `paper_size`, `orientation` |
| `test_page` | Render and print a timestamped test page (printer name, state, time) |
| `print_text` | Render and print the `content` field as plain text |
| `print_image` | Print a generated image from a named slot (`canvas`, `portrait`, `wallpaper`, `id_pic`, `dashboard_snap`, `home_state`) — reads from `/config/www/zen_images/<slot>.jpg` via the local HA `/local/` path |
| `status` | One answer from every source the printer has: state and job state, `paper_empty` (from the IPP paper reason), ink levels with the printer's own low threshold (`ink_low`), and, with `epson_workforce`, `maintenance` (maintenance-box level, page counters, scanner status). `sources` says where each part came from. `printer_ip` comes from the printer's own IPP address. Reads HA entity state only — no IPP call |
| `brief` | 3D printers, read-only. A one-line headline with a level (`alert`, `attention`, `done`, `running`, `idle`) plus ready `postman` and `camera` payloads, and the job-end `steps` that are on. Sends nothing |
| `filament` | 3D printers, read-only. The printer's own estimate for the job (grams, meters), the AMS spool context, what Inventory has for filament, and a proposed consumption entry for a person to confirm. Never changes stock |
| `summary` | The `print_shop` component's seed: the 3D printer (headline, level, alarms, control, the camera's last description) and every paper printer (paper, low ink), each scored with `urgency`, `action_required` and real entity ids. Needs `zenos.media.print_status` |
| `kfc_manifest` | Returns the `print_shop` component definition |
| `printers` | Every printer with its `kind` (`epson_ipp`, `fdm_bambu`, `fdm_octoprint`), area, labels and companion sensor count; for a 3D printer, whether control is available. Read-only |
| `readiness` | 3D printers only, read-only. Device checks (online, idle, no alarms, door, SD card, filament loaded and identified, camera) plus `human_checks_always_needed` (clear build plate, someone watching, filament matches the job). `can_start_remotely` needs a ready printer and control |
| `list_jobs` | All IPP jobs (`job_filter`: `all` \| `incomplete` \| `complete`) |
| `clean` | Trigger a head-cleaning cycle via the printer's HTTP maintenance UI (~3 minutes) |
| `nozzle_check` | Trigger a nozzle-check pattern via the printer's HTTP maintenance UI |
| `help` | Field reference + examples. A bare call (no mode) returns help |
| `tool_manifest` | Standard manifest response |

**A bare call returns help.** Ask for `mode=status` by name.

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
- **`clean` and `nozzle_check` use the single `epson_ip` in the cabinet,** not each printer's own address, so they reach only the configured printer.
- Single-printer defaults are baked into the cabinet drawer; use `printer_entity` per-call for anything beyond the configured default.

---

## Examples

```yaml
# One-time setup
zen_dojotools_print_shop:
  mode: configure
  epson_ip: 192.0.2.10
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
| 0.6.4-experimental | The component subscribes to `print_done` while `job_end_summary` is on; Postman is suggested only when `notify_on_job_end` is on; a 3D `finish` counts as done for 12 hours. |
| 0.6.3-experimental | The `print_status` cert names its mode (`modes: ['summary']`), so a live drawer on `summary` is allowed by the cert. |
| 0.6.2-experimental | `zenos.media.print_status` allows stage `l`. |
| 0.6.1-experimental | Job-end camera look, summary and Postman push are settings, off by default. |
| 0.6.0-experimental | Kung Fu component host: `kfc_manifest`, the `summary` seed, `zenos.media.print_status`. |
| 0.5.0-experimental | `brief` and `filament` modes (3D, read-only). |
| 0.4.1-experimental | A bare call answers help, not status. |
| 0.4.0-experimental | Read-only 3D printer support (Bambu Lab): `printers` reports `kind` and control, `status` returns a 3D digest, new `readiness` mode. Nothing is sent to a 3D printer; control fails closed. AMS slots the printer cannot identify are not counted as loaded. |
| 0.3.0-experimental | `status` merges IPP and `epson_workforce` for the same printer (matched by device model): `paper_empty`, `ink_low`, `maintenance`, `sources`. No entity names are hardcoded, so a second printer works the same way. `printer_ip` from the printer's IPP address. Default printer must have an IPP address. New read-only `printers` mode. |
| 0.1.1-experimental | Printer falls back to the `printer`-labeled sensor when `epson_entity` isn't configured, instead of a fixed entity ID. |
| 0.1.0-experimental | Initial release. Text + image printing via IPP/Rudd-O, WF-7840-specific maintenance triggers. |
