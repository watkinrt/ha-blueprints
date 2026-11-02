# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A collection of Home Assistant **blueprints** — declarative YAML automation
templates imported into HA, not an application. There is no build step, no test
suite, and no dependencies to install. Each top-level `.yaml` file is one
standalone, self-contained blueprint.

Current contents: `reolink_notifications.yaml` — Reolink camera detections →
Telegram clips/snapshots.

## Validating changes

There is no local linter or runner. Verification happens in Home Assistant:

```bash
# Catch YAML syntax errors before pushing
python3 -c "import yaml,sys; yaml.safe_load(open('reolink_notifications.yaml'))"
```

Jinja templates inside the blueprint can only be checked in HA's
**Developer Tools → Template** editor (substituting concrete values for
`!input` variables), and the blueprint itself via
**Settings → Automations & Scenes → Blueprints → Import** (re-import to pick up
changes; existing automations built from the blueprint inherit edits).

Blueprints are imported by raw GitHub URL, so a push to `main` is effectively a
release. The repo is published at `github.com/watkinrt/ha-blueprints`.

## Blueprint conventions used here

- **`!input` is resolved before templating.** An `!input` cannot appear inside a
  Jinja expression. The established pattern is to alias every input once in the
  `variables:` block (`camera_entity: !input camera_entity`), then use only
  those names in templates. Keep this — it also makes the input list readable in
  one place.
- **Derived values belong in `variables:`**, not inlined at the call site
  (`want_clip`, `cam_slug`, `caption`, `ffmpeg_vf`, `live_url`,
  `live_keyboard`). Each derived variable carries a short comment explaining the
  *why*, not the syntax.
- **`mode: single` + `max_exceeded: silent` is load-bearing.** The cooldown is
  implemented as trailing `delay` / `wait_template` steps at the end of
  `actions:`, so the run staying alive *is* the cooldown — re-triggers are
  dropped as "already running". Changing the mode silently removes rate
  limiting.
- **`continue_on_error: true`** on every step that touches the outside world
  (`camera.record`, `shell_command.*`, `camera.snapshot`, the first Telegram
  send) so a failure degrades to the next fallback instead of aborting the run.
  Clip failure paths are the `choose:` block in step 3.
- **Descriptions are user-facing documentation.** Input `description:` fields
  and the blueprint `description:` are what someone reads in the HA UI; they
  carry the setup instructions (including the required `shell_command:` snippet
  for `configuration.yaml`) and the rationale for recommended values. Treat them
  as part of the deliverable when changing behaviour.
- **Inputs are grouped into collapsible sections** (`camera_settings`,
  `style_settings`, …) each with a `name` and `icon`. New inputs go into an
  existing section rather than at the top level.

## External contracts to be careful with

- **`shell_command.reolink_clip_to_animation`** must exist in the user's
  `configuration.yaml` — the blueprint cannot create it. Its parameter names
  (`src`, `dst`, `vf`) are a contract with the snippet in the blueprint
  description; changing one requires changing both.
- **Telegram inline keyboards** take `"Label:url"` strings, require `https`, and
  break on commas in the URL — hence the guard in `live_keyboard`.
- **`telegram_bot.send_photo` response** (`snap_resp.chats[].message_id`) is only
  usable for `reply_to_message_id` when there is exactly one target chat, since
  message ids are per-chat.
- **`lookback:`** on `camera.record` silently does nothing unless "Preload
  stream" is enabled on the camera entity — `Stream.async_record` reads it from
  the HLS output, which only exists while something keeps the stream alive. It
  is also quantised and capped: `min(int(lookback / ~2s) + 1, MAX_SEGMENTS=5)`
  segments, minus the newest, so ~8 s is the ceiling whatever the input says.
- **`telegram_bot.*` sends block until the upload completes.** Anything latency
  sensitive must not sit behind one — hence the `parallel:` block holding the
  snapshot branch and `camera.record` in step 2.
- **A `response_variable` set inside a `parallel:` branch IS visible after the
  block**, contrary to the usual advice. `ScriptRunVariables._assign` recurses
  into the parent scope; only `assign_parallel_protected` (`wait`) and
  `define_local` (`repeat`) are branch-local. This is what lets the clip still
  reply to the snapshot's `message_id`.
- **Working folder** must be writable by HA by default (`/media` on Docker,
  `/config/media` otherwise) or listed in `allowlist_external_dirs`;
  `/config/www` is served unauthenticated at `/local/` and is deliberately
  discouraged.
