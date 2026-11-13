# Dance Routine — notes

A single-file app in `index.html` for laying out a first dance against a song: the steps sit on a
timeline, light up as the song plays, and a rehearsal video can play in sync. No build step, no
dependencies (the QR code library loads from cdnjs only when the share dialog opens).
Live at https://candlebird.github.io/dance-routine/ (every push to `main` redeploys it to GitHub Pages).

## Running locally

`python -m http.server 8765 --bind 127.0.0.1`, then open http://127.0.0.1:8765/index.html.
Port 8765 on 127.0.0.1/localhost is in the wedding API's CORS list, so online saving works locally too.
Opening `index.html` straight from disk works, but without online saving.

## Data

- The routine autosaves to localStorage (`firstDancePlanner.v1`). The song and video are kept in
  IndexedDB (`firstDancePlanner` → `kv` → `audio` / `video`, each `{blob, name, mediaId}`).
  **Export** / **Import** move the routine (not the media) as a `.dance.json` file.
- **☁ Save online** stores the routine in the wedding API (the sibling `WeddingApi` Cloudflare Worker,
  kind `dance`) and uploads the song and video to its R2 bucket (`wedding-media`, as `<docId>/<mediaId>`).
  `routine.media` holds `{song, video}` = `{id, name, type, size}` of the uploaded files.
  - The link is `…/#edit=<id>.<key>`. The key is only in that link and in localStorage (`danceRoutine.cloud`).
    Anyone with the link can open and edit.
  - Changes save about 1.5 s after the last edit. Other devices check every 20 s and when the tab comes back
    into view. If another device saved first, theirs loads and Undo brings yours back (Undo never changes
    which song/video is attached).
  - Loading a new song or video while online uploads it in 8 MB parts (multipart, so videos over the Workers
    100 MB request limit work), then the old file is deleted. The header button shows progress.
  - On another device the song is downloaded once (it's needed for the waveform) and cached in IndexedDB.
    The video streams straight from the API (Range requests), so it starts playing right away.
- Media limits (in the worker): 2 GB per file, 8 files per routine, 9 GB for the whole bucket (R2 free tier is 10 GB).

## Features

| Feature | Notes |
|---|---|
| Timeline | Waveform, beat grid, bar markers, step blocks, playhead. Drag steps / edges, Ctrl+scroll zoom. |
| Beat grid | BPM + "beat 1" offset, or hand-placed bars (`routine.bars`, the time of each "1"). Beats are spread evenly within each bar, so drifting tempos work. |
| Shift / stretch | Moves or scales steps and/or bars together; "fit last step to" a time. |
| Rehearsal video | `routine.video = {start, rate, volume, mirror, size, bpm}`. Video time = (song time − start) × rate. Silent by default. Sync points pick start and rate from two matching moments. |
| Practice | Speed 50–100% (pitch kept), loop a step with lead-in, practice view (F). |
