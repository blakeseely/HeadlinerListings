# HeadlinerListings — show-data feed for the Headliners iOS app

One `shows.json` per venue (19 venues) + a `manifest.json` the app fetches
first. Every file is regenerated nightly by the headliners pipeline
(`~/.hermes/projects/headliners`) from each venue's scraped calendar.

**Current schema: `4`.** Every venue file carries a `schema` integer
(incremented on any shape change) and a `field_hints` map. Gate your parsing
on `schema`, and on `field_hints` for per-field semantics.

## Layout

```
manifest.json          venues: venue_id, file, city, metro, updated_at,
                       oldest_show_date, show_count, sha256_16
<venue>/shows.json     venue, venue_id, venue_url, source_url, city, metro,
                       neighborhood, schema, field_hints, scraped_at, count,
                       shows[], stats
```

### manifest.json fields
| field | meaning |
|---|---|
| `generated_at` | when this manifest was written (ISO 8601, UTC) |
| `venues[].updated_at` | **when this venue's data was last refreshed** (= the file's `scraped_at`) |
| `venues[].oldest_show_date` | oldest show date in the file (the pre-rename meaning of `updated_at`; kept for old clients) |
| `venues[].sha256_16` | first 16 hex of the file's SHA-256 — verify before use |
| `venues[].value_domains` | the controlled vocabulary actually present in this file (see below) |

## Venue file: top level
| field | hint | notes |
|---|---|---|
| `venue` / `venue_id` / `venue_url` / `source_url` | string | identity. The six Belly Up Presents San Diego venues intentionally share `venue_url: https://bellyup.com` (group calendar) |
| `city` / `metro` / `neighborhood` | string \| null | geo tags; `metro` is one market value ("San Francisco" / "San Diego" / "Los Angeles") |
| `schema` | int | **4** — the contract version this file speaks |
| `field_hints` | object | per-field controlled-vs-raw contract (below) |
| `scraped_at` | date | when this file was built (the refresh date) |
| `count` | int | `len(shows)` |
| `shows` | array | sorted ascending by date; no duplicates within a file |

## Venue file: show record (18 keys, every show)
| field | hint | vocabulary |
|---|---|---|
| `event_type` | **controlled** | `music` \| `comedy` \| `film` \| `other` \| null |
| `needs_review` | bool | bill-level flag (e.g. 3+ co-headliners); **independent** of artist `flagged` in stats |
| `headline_artists` / `support_artists` | artist[] | see below |
| `multi_artist` | bool | |
| `show_title` | free string \| null | tour/show name when billed separately |
| `presenters` | string[] | "X Presents" |
| `set_format` | free string \| null | "Live + DJ" style |
| `date` | date \| null | `YYYY-MM-DD` |
| `doors` / `show_time` | **free string** \| null | canonicalized to `H:MM pm` **when parseable** (e.g. the venue's `7 PM` and `8:00PM` both become `8:00 pm`); unparseable text passes through verbatim |
| `ticket_link` | free string \| null | usually a URL, but some venues publish button text ("Buy Tickets"); sniff `http` if you need URLs only |
| `ticket_status` | **free string** \| null | case/separator duplicates are folded (`On Sale`→`on sale`, `on-sale-soon`→`on sale soon`); all other wording is the venue's raw text. Observed: `on sale`, `sold out`, `not on sale`, `on sale soon`, `presale`, `coming soon`, `waitlist`, `cancelled` |
| `event_page` | string | the venue's own page for this event |
| `room` | free string \| null | multi-room venues |
| `age_rating` | **free string** \| null | the venue's own text, verbatim (15+ spellings of the same concept exist: `all ages` / `All Ages` / `All ages` / `5 and Over` …) — normalize client-side if you need a fixed set |
| `note` / `description` | free string \| null | |

## Artist record (8 keys, **always all present** — one shape)
| field | hint | meaning |
|---|---|---|
| `name` | string | as billed |
| `apple_music` | url \| null | Apple Music artist link — **present iff a confident match was found** (this is the found/not-found signal) |
| `spotify` | url \| null | Spotify artist link |
| `confidence` | number \| null | Apple match confidence: `null` = match not attempted (film/director names, censored acts), `0.0` = attempted, no match, else `0–1` |
| `matched` | string \| null | the catalog artist name the link resolves to (retained for below-floor near-misses too) |
| `apple_label` | free string \| null | Apple review label (`close match: X`, `compound bill — links to A only`, `no confident match …`) |
| `spotify_label` | free string \| null | Spotify review label, same styles |
| `spotify_source` | **controlled** \| null | `venue` (venue page published it) \| `api` (Spotify Web API lookup) \| `feed` (venue's structured feed) \| null (no link) |

## stats (uniform in every file: `event_types`, `apple_music`, `spotify`)
| block | identity clients may rely on |
|---|---|
| `event_types` | `{music, comedy, film, other}` — **all four keys always present**, zeros included. Un-typed shows count in none |
| `apple_music` | per **unique** artist (deduped case-insensitive on name): `found == exact + close + flagged`; `unique_artists == found + not_found`. `exact` = conf 1.0; `close` = conf < 1.0, no label; `flagged` = carries an `apple_label` (human should eyeball). `flagged` is match-quality only — NOT the same thing as show `needs_review` |
| `spotify` | per unique artist: `linked == venue + api + feed`; `shows_with_link` = shows where **any** billed artist has a link |

## Semantics worth knowing (deliberate decisions, not accidents)
- **Link presence IS the match signal.** An artist has an Apple Music link
  iff a confident catalog match was found (below the 0.80 link floor it is
  honest null + an `apple_label`). An artist with no Apple link may still
  carry a Spotify link (24 of 1,104 do). For "does this act have *any*
  link", test `apple_music != null || spotify != null`.
- **No derivable convenience fields.** The feed carries no `date_label` (was
  a pre-formatted date — derive `"Www YYYY-MM-DD"` from `date`) and no
  `match_label` (was the `apple_label`/`spotify_label` join — join
  client-side if you want one display string). Everything in the file is a
  *source* of truth, not a projection of another field.
- **Controlled vs raw.** The `field_hints` map says which fields are
  controlled (safe to `switch` on) and which are raw venue strings
  (normalize client-side). The manifest's `value_domains` lists the controlled
  values actually present in each file.
- **A show can appear under two venues.** E.g. strongboi 2027-03-09 is listed
  by both Castro and The Independent: the Independent's page is a
  TicketMaster embed for the Castro show (`MOVED TO CASTRO THEATRE`), so both
  files legitimately carry it — same ticketing event, two calendars.
- **Off-season venues are valid at zero.** `frost` (Stanford) has no winter
  calendar: `count: 0`, zeroed stats.

## Value domains (schema 4)
Controlled vocabularies actually in the feed right now (the manifest carries
a per-file `value_domains`; these are the union):

| field | domain |
|---|---|
| `event_type` | `music`, `comedy`, `film`, `other`, null |
| `ticket_status` | `on sale`, `sold out`, `not on sale`, `on sale soon`, `presale`, `coming soon`, `waitlist`, `cancelled` (free string — new venue wording passes through) |
| `doors` / `show_time` | `H:MM pm` when parseable; other venue text verbatim |
| `age_rating` | free string (raw venue text) |
| `artist.confidence` | null / 0.0 / fractional 0–1 |
| `artist.spotify_source` | `venue`, `api`, `feed`, null |
| `metro` | `San Francisco`, `San Diego`, `Los Angeles` |

## Schema history
- **1** — original 19-venue shape; 8/7-key artist split.
- **2** (2026-10-23) — uniform 10-key artist; stats `needs_review` → `flagged`;
  manifest `updated_at` = refresh date (`oldest_show_date` kept); `field_hints`
  added.
- **3** (2026-10-23) — removed the per-artist `found` key (redundant with
  `apple_music`).
- **4** (2026-10-23) — removed `date_label` (show) and `match_label` (artist);
  both are pure projections of other fields. Shows are now 18 keys, artists
  8 keys.
