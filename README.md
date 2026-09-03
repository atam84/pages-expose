# sudoku-cube.virt-services.com

Static site for **Sudoku Cube** (Android, `com.virt.sudokucube`), published by
virt-services. Served through Cloudflare — static files only, no server-side code.

## Layout

```
index.html                  Landing page (features, support, FAQ)
privacy-policy/index.html   Privacy policy — linked from app.config.ts and the Play listing
app-ads.txt                 AdMob authorized-seller declaration
daily/                      Daily Cube puzzle payloads (see below)
  index.json                Manifest: available dates, key id, algorithm
  <YYYY-MM-DD>.json         One signed DailyPuzzleEnvelope per UTC day
                            Currently 2026-09-01 .. 2027-10-05 (400 days)
api/
  index.json                Service index
  leaderboard/index.json    Leaderboard contract — status: not-implemented
```

## Daily CDN — `/daily/<YYYY-MM-DD>.json`

The app fetches `${dailyCdnBase}/daily/${date}.json` where `dailyCdnBase` is
`https://sudoku-cube.virt-services.com` (`app.config.ts` → `extra.dailyCdnBase`).
Each file is a `DailyPuzzleEnvelope` signed with HMAC-SHA-256 (`keyId: k1`) per
ADR 0006. The client verifies the signature before accepting a puzzle and refuses
tampered files.

A missing date is a plain 404. That is not an error condition for the app: the
loader falls back to its on-device cache, then to generating the day's puzzle
locally, so Daily play never depends on this directory being current.

That fallback reproduces these files exactly rather than substituting a
different puzzle. `src/services/daily/localGeneration.ts` holds the seed, step
budget and retry ladder, and the generation script below imports the same
module — verified across all 400 published dates. Change a constant in that
file and every file here must be regenerated, or offline players start seeing a
different board from online ones.

| Constant | Value | Why it is what it is |
| --- | --- | --- |
| `DAILY_BASE_SEED` | `20260901` | Arbitrary but frozen — every file here was built from it. |
| `DAILY_MAX_STEPS` | `50000` | Solver step cap per uniqueness check. Under the solver's default 10 M ceiling a pathological seed can run for tens of seconds (2027-04-17 measured at 39 s in Node), which on-device would freeze the JS thread. |
| `DAILY_MAX_ATTEMPTS` | `8` | Deterministic seed ladder. A seed that blows the step budget fails fast and the next one is tried. 4 sufficed for all 1095 dates measured; 8 is headroom. |

Measured over three years of dates: median 45 ms, worst case 268 ms in Node.

### Regenerating

From the app repo, with the signing key that matches the shipped build
(`EXPO_PUBLIC_DAILY_HMAC_KEY` in `app.config.ts` / `.env.local`):

```bash
DAILY_PUZZLE_HMAC_KEY=<64-hex-chars> \
  npx ts-node --compiler-options '{"module":"commonjs"}' \
  scripts/generate-daily-puzzles.ts \
  --start 2026-09-01 --days 400 --out website/daily/
```

The seed comes from the date, so there is no `--seed` flag — passing one exits
2 rather than being ignored, because a caller-chosen seed would silently
desynchronise these files from what devices generate offline. A regenerated
date therefore reproduces byte for byte. Existing files are skipped, so
re-running only fills gaps; extend coverage by raising `--days`. Then verify:

```bash
DAILY_PUZZLE_HMAC_KEY=<64-hex-chars> \
  npx ts-node --compiler-options '{"module":"commonjs"}' \
  scripts/verify-daily-puzzles.ts --dir website/daily
```

`verify-daily-puzzles.ts` re-derives each HMAC and re-checks each puzzle, so it
catches both a corrupted file and a key mismatch.

`daily/index.json` is **not** written by the generator — refresh it after adding
dates, or it will under-report what is published. It is a convenience manifest
for humans and tooling; the app never reads it, it requests dates directly.

Rotating the key means regenerating every file here **and** shipping an app
update with the new key — old builds cannot verify new files. See ADR 0006.

## Leaderboard — `/api/leaderboard`

**Not implemented.** There is no leaderboard server, and static hosting cannot
accept a POST — `/api/leaderboard/submit` returns 404. `api/leaderboard/index.json`
records the intended request/response contract so the client and a future backend
agree on it.

The app treats 404/405/410/501 from this path as *leaderboard unavailable*: it
hides the leaderboard panel on the daily completion screen, drops any queued
submission, and stops asking for the rest of the session. Nothing about a
missing leaderboard blocks play, and no request is retried against an endpoint
that cannot answer.

Transient 5xx responses are deliberately *not* in that set — a real backend
having a bad moment keeps its retry behaviour.

App-side: `src/services/leaderboard/client.ts` maps the status codes,
`src/state/leaderboardStore.ts` holds the session latch that stops re-asking,
and `src/components/daily/LeaderboardPanel.tsx` renders nothing when
unavailable. Note the base URL already carries the `/api/leaderboard` prefix, so
the client appends only `/submit`.

Standing this up for real needs a dynamic host (Cloudflare Workers/Pages Functions
or similar) behind the same domain, not this repo.

## Contact

support@virt-services.com
