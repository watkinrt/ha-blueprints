# ha-blueprints

Home Assistant blueprints I use at home. Each file is a standalone blueprint —
import the one you want, no other setup in this repo required.

| Blueprint | What it does |
| --- | --- |
| [`reolink_notifications.yaml`](reolink_notifications.yaml) | Posts a Reolink camera clip or snapshot to Telegram on detection, with a per-camera cooldown. |

---

## Reolink clip to Telegram

When a camera's detection sensor turns on, this posts to one or more Telegram
chats: a caption line (`🎥 Front Door — Person · Fri 7:12:04 PM`), the footage,
and an optional **📹 Live view** button that opens the camera in Home Assistant.

[![Open your Home Assistant instance and show the blueprint import dialog.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fwatkinrt%2Fha-blueprints%2Fblob%2Fmain%2Freolink_notifications.yaml)

Requires Home Assistant **2025.7.0** or newer.

### Three notification styles

| Style | Behaviour | Good for |
| --- | --- | --- |
| **Clip** | Records, converts, posts one silent auto-looping clip. Falls back to a snapshot if the clip fails. | Most cameras. |
| **Snapshot, then clip** | Posts a still immediately, then posts the clip as a reply to it. | Places where seconds matter — the front door. |
| **Snapshot only** | One still, no recording. Fastest. | Low-value triggers, slow hardware. |

The clip is sent as a Telegram *animation*, so it loops silently in the chat
rather than needing a tap to play.

### Setup

**1. Add the shell command.** The blueprint shells out to `ffmpeg` to convert
the camera recording into something Telegram will loop. Blueprints can't define
shell commands, so this goes in `configuration.yaml` once, then restart HA:

```yaml
shell_command:
  reolink_clip_to_animation: >-
    ffmpeg -y -loglevel error -i {{ src }} -an -vf {{ vf }}
    -c:v libx264 -preset veryfast -crf 28 -pix_fmt yuv420p
    -movflags +faststart {{ dst }}
```

Skip this step if you only ever use **Snapshot only**.

**2. Set up the Telegram bot integration** with your group chat as an allowed
chat. It creates one `notify.*` entity per allowed chat; those are what you pick
in the **Telegram chats** option.

**3. Create the automation** from the blueprint, once per camera.

### Choosing the camera entity

Pick the **`_fluent`** (sub-stream) entity where you have one. It's lower
resolution and H.264, so recording and conversion are fast and the resulting
clip is small. The main stream works but is slower and much larger for no real
benefit in a chat window.

If you want **Lookback** (footage from *before* the trigger), enable
*Preload stream* in that camera entity's settings — without it, lookback
silently does nothing. It's worth it for **Snapshot, then clip**, where
recording starts a second or two after the event.

### Trigger sensors

Point **Trigger sensors** at whichever binary sensors you care about —
`_person`, `_vehicle`, `_animal`, `_visitor`, `_motion`. Any of them turning on
starts an event; the caption names the detection type based on which one fired:

`_visitor` → *Doorbell pressed* · `_person` → *Person* · `_vehicle` → *Vehicle* ·
`_animal` / `_pet` → *Animal* · `_package` → *Package* · `_face` → *Face* ·
anything else → *Motion*

### Cooldown

Each automation is one camera, and only one run happens at a time — triggers
arriving mid-run are dropped silently. The run itself is the cooldown:

1. post the notification, then
2. wait out **Minimum cooldown** (default 60 s), then
3. if **Also wait for sensors to clear** is on, wait until *every* trigger
   sensor is off, capped by **Max wait for clear** so a stuck sensor can't
   wedge the automation forever.

So a car sitting in the driveway with `_vehicle` held on produces one message,
not one a minute.

### Live view button

Telegram only accepts `https` links on buttons, so the button is hidden unless
**Home Assistant external URL** is an `https` address
(`https://xxxx.ui.nabu.casa`, `https://ha.example.com` — no trailing slash).

Also set **Live view page** to a dashboard view holding this camera in a
*picture-entity* card with `camera_view: live`, e.g.
`/dashboard-cameras/front-door`. That card autoplays muted, so the button gives
you live footage in one tap. Left blank, the button opens the camera's more-info
pop-up instead — which browsers won't autoplay, so you have to press play.

### Working files

The clip and snapshot are written to **Working folder** (default
`/media/camera_clips`) and **overwritten on every event** — history lives in the
Telegram chat, not on disk. HA creates the folder itself, but it has to sit
somewhere HA may write: `/media` on Docker installs, `/config/media` otherwise,
or any path in `allowlist_external_dirs`.

Avoid `/config/www`: everything in it is served at `/local/` with no login.

### Other options

- **Clip length** (3–30 s), **width**, **frame rate**, and **playback speed**
  (1× / 1.5× / 2× — faster clips are quicker to skim later).
- **Send silently** posts without a notification sound. The clip in
  *Snapshot, then clip* is always silent, since the snapshot already alerted
  everyone.
- **Camera label** overrides the name in the caption; blank uses the camera's
  friendly name.
- **Snapshot delay** (0–5 s) — 0 is fastest, 1–2 s sometimes frames the subject
  better.
- **Pause toggles** — if any listed `input_boolean` / `switch` / `binary_sensor`
  is on, nothing is posted. Handy for an "everyone's home" helper.
- **Additional conditions** — any standard HA conditions, e.g. only after dark.

### If something doesn't arrive

Every outward step tolerates failure, so the automation degrades instead of
dying. That also means failures are quiet — check the automation's trace first.

- **No clip, snapshot instead** — `ffmpeg` failed or isn't on `PATH` inside the
  HA container. The trace shows the shell command's `returncode` and `stderr`.
- **Nothing at all** — a pause toggle is on, an additional condition blocked it,
  or the automation is still inside a previous cooldown.
- **Clip isn't a reply to the snapshot** — expected with more than one target
  chat; Telegram message ids are per-chat, so threading only works with exactly
  one.
- **No Live view button** — the external URL isn't `https`, or the resulting URL
  contains a comma, which HA's keyboard parser treats as a separator.
