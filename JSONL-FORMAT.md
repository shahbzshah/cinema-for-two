# Cinema for Two — JSONL memory format

Memories are an append-only JSONL log, one JSON object per line, same pattern
as the crash bot's `signal-history-archive.jsonl`. No database, no accounts.
The log lives in `localStorage` under `c2t-log-<ROOM CODE>` and can be
downloaded / loaded as `cinema-for-two-<CODE>-memories.jsonl` — the file
IS the backup and the carrier.

## Line shape

```json
{ "id": "k3f9xq2m7a1b", "v": 1, "type": "media", "room": "ABC123", "at": 1763740000000, "by": "c2t-user-a1b2c3d4", "byName": "Alex", "data": { ... } }
```

| Field | Meaning |
|---|---|
| `id` | Random 12-char id. Merging dedupes on this — the same line never counts twice. |
| `v` | Format version, currently `1`. |
| `type` | What happened (see below). |
| `room` | 6-char room code. Lines for another room are ignored on import/merge. |
| `at` | Epoch milliseconds. Replay order + "newest wins" use this. |
| `by` | Local user id (`c2t-user-<8 chars>`), generated once per browser, stored in `localStorage` under `c2t-user-id`. |
| `byName` | Display name at the time of the line. |
| `data` | Payload, shape depends on `type`. |

## Types

| Type | `data` shape | Replay rule |
|---|---|---|
| `media` | `{ type: "youtube" \| "url" \| "local", url, title, localFileName }` | Last line wins. (`local` shares only the filename — both partners pick the same file on their own device.) |
| `playback` | `{ isPlaying, currentTime, actionLabel, seq }` | Last line wins. Saved on play, pause, seek, media load, every 10s while playing, and on page close. |
| `whisper` | `{ text, movieTimestamp, senderName }` | Accumulates, sorted by `at`. |
| `rating` | `{ movieTitle, submission: { uid, name, score, vibes, note, submittedAt } }` | Accumulates, one submission per user (latest per `uid` wins). |
| `rating-clear` | `{}` | Clears the rating session (written when a passport stamp spends it). |
| `passport` | `{ entry: { id, movieTitle, coupleAverage, ratings, vibes, watchedDate, createdAt } }` | Accumulates, newest first. |
| `passport-delete` | `{ entryId }` | Tombstone — the matching `passport` entry is hidden on replay. The entry line stays in the log. |

NOT logged (live-only, over PeerJS): reactions, presence, ready-check.

## Example lines

```json
{"id":"m1a2b3c4d5e6","v":1,"type":"media","room":"ABC123","at":1763740000000,"by":"c2t-user-a1b2c3d4","byName":"Alex","data":{"type":"youtube","url":"dQw4w9WgXcQ","title":"YouTube Feature","localFileName":""}}
{"id":"p7q8r9s0t1u2","v":1,"type":"playback","room":"ABC123","at":1763740065000,"by":"c2t-user-a1b2c3d4","byName":"Alex","data":{"isPlaying":false,"currentTime":382.5,"actionLabel":"paused","seq":4}}
{"id":"w3x4y5z6a7b8","v":1,"type":"whisper","room":"ABC123","at":1763740080000,"by":"c2t-user-e5f6g7h8","byName":"Sam","data":{"text":"Wait rewind that!","movieTimestamp":"06:22","senderName":"Sam"}}
```

## Rules

- **Merge:** by `id`. Unknown / corrupt lines are skipped on import (the app toasts how many loaded vs skipped).
- **Replay:** lines are sorted by `at`; media/playback = last wins; whispers/ratings/passport accumulate; `passport-delete` tombstones hide entries; `rating-clear` resets the session.
- **Sync:** on connect, both sides swap their full logs over PeerJS and merge by `id`. Every live change is also a log line sent to the partner, so both logs converge.
- **Backup:** "Download memories (.jsonl)" in the Movie Passport card. Load a file from the welcome screen (it fills in the room code) or from the Passport card.
- **Limits:** a 2-hour night is a few hundred lines (~100 KB). If BOTH phones lose their browser storage AND the .jsonl file, the memories are gone — there is no cloud copy. That is the honest trade-off of this step; a database comes later.
