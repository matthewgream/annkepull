# annkepull

Archives the event clips that Annke cameras record to their own SD cards. Runs daily from a systemd
timer; each run fetches every clip on each card that is not in the archive yet, so the first run
backfills whatever the cards still hold and later runs pick up just the new ones.

Clips are stored as `<archive>/<camera>/<YYYYMMDD>/recording_<YYYYMMDDHHMMSS>.mp4` (local start
time, file mtime = clip start), remuxed from the camera's own container -- video is not re-encoded.
Day folders keep directory listings small (front alone records 50-70 clips a day). A second
clip starting in the same second (the camera sometimes splits an event across its storage files)
gets a `_1` suffix.

| | |
|---|---|
| `annkepull` | the tool (Node, no npm deps; uses `curl`, `ffmpeg`, `ffprobe`) |
| `annkepull.timer` | runs `annkepull.service` daily at 04:00 (+ up to 30 min), catching up after downtime |
| `annkepull-storage.service` | gate: the run needs `opt-storage-disk.mount` |

## Install

```sh
make install      # tool, config, storage gate, timer
make run          # a run now, in the background (the first one backfills: hours)
make status       # timer, last run, last log lines
make test         # dry run: what the next run would fetch
```

Config goes to `/etc/default/annkepull` (mode 600: it holds the camera password). A per-host
`annkepull.<hostname>.cfg` is used instead of `annkepull.cfg` when present.

## How it works

The cameras run Hikvision firmware, so their recordings are reachable over ISAPI (HTTP, digest
auth):

- `POST /ISAPI/ContentMgmt/search` on track 101 (main stream) lists every clip on the card with its
  start/end (UTC) and a playback URI, 50 per page.
- `GET /ISAPI/ContentMgmt/download`, with the playback URI in an XML body, returns the clip: an
  `IMKH` header around MPEG-PS, which `ffmpeg -c copy` turns into mp4.

There is no state file: a clip counts as archived when its file exists, so an interrupted or failed
run is completed by the next one (files are written as `.part` and renamed when complete). Clips are fetched oldest first,
since a full card overwrites its oldest clips. Clips that ended less than `settle` seconds ago are
left for the next run, as the camera may still be writing them. After a failed download the run
pauses (1, 2, 3, 4 minutes for successive failures) since cameras sometimes return HTTP 500 for a
short while; a camera that fails 5 downloads in a row is left for the next run.

## Notes

- The search's `Z` times are real UTC (checked against a known event), not camera local time -- except
  across a DST change on some firmware, see below.
- Some firmware (front: V5.7.1 build 230627) ends a search at 1000 results and reports it complete,
  so the card is listed by searching again from the last clip found until nothing new turns up.
- The same firmware converts clip times to UTC with the camera's offset *now*, not the one when the clip
  was recorded. So after a DST change every clip from the other half of the year is reported an hour
  out. On 2026-10-25 that re-fetched front's whole card under names an hour late.
  `times-<name>=current-offset` in the config undoes it: annkepull reads the camera's current offset
  (`/ISAPI/System/time`) and adds it back. side (build 220316) reports true UTC. To check a camera,
  compare a clip's on-screen clock with its file name, in both halves of the year.
- The download is a GET with a body, which `fetch()` will not send -- hence curl.
- A camera without an SD card (e.g. `wide`) has nothing to pull; leave it out of the config.

## License

CC BY-NC-SA 4.0 — see [LICENSE](LICENSE).
