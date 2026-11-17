# SFX Analysis Format: specification v0.1 (draft)

Status: DRAFT, 2026-11-17. Fields, units and positions are documented from the reference analyser's code; the reference analyser is published when it passes the known-answer set (section 13). Feedback welcome through issues.

## 1. What this is

- A plain-text format that stores measurements of every sound-effects file in a library. Any AI agent or deterministic tool can then build a strong starting pool of sounds without decoding the audio again.
- It holds facts about each file: levels, timbre, envelope, every hit's position, space, stereo and loudness. None of these depend on how the file will be used.
- It holds NO audio, and nothing that can rebuild audio. The numbers are aggregates, positions and short curves.
- It does not choose sounds. It feeds a pool of suggestions. Names and categories (UCS) say WHAT a sound is; this format says HOW it sounds and WHERE its hits are. Use both.
- Each user analyses their own library. The specification and the analyser are public. No index of anyone's library is ever part of the release.

The key words MUST, SHOULD and MAY are used as in RFC 2119.

## 2. Terms

| Term | Meaning |
|---|---|
| file sample | A sample position in the audio file itself, at the file's own sample rate, counted from 0. All stored positions are file samples unless the field name ends in `Ms`. |
| `...Ms` | Milliseconds from the start of the file (not of a hit), unless the field says otherwise. |
| listening view | The analysis signal: the first 600 s of the file's first audio stream, mixed to one channel and resampled to 32,000 Hz, as 32-bit float (not clipped). |
| block | 1 ms of the listening view = 32 samples. |
| 5 ms level (E5) | dBpow of the mean square over the 5 blocks centred on a block. |
| dBpow | `10*log10(mean square + 1e-14)` of the listening view. 0 dBpow = DC at full scale; a full-scale sine reads -3.01; -140 = digital silence. Used for levels. |
| dBpk | `20*log10(|sample| + 1e-12)` of the listening view. Sample peak; can be above 0 because the view is float. |
| dBamp | `20*log10(RMS)` of the 48 kHz FRONT copy. Used only for `tailFloorDb`. |
| LUFS / LU / dBTP | ITU-R BS.1770-4 loudness, EBU Tech 3342 loudness range, 4x-oversampled true peak, measured on all channels at the file's own rate (describe layer only). |
| floor | The 5th percentile (lower, no interpolation) of all 5 ms levels. |
| sound line | `min(peak5ms - 6, max(peak5ms - 40, floor + 10))` dBpow. A level the sound is "on" above. |
| attack | A rise of 12 dB or more over the lowest 5 ms level of the previous 31 ms, at or above the sound line, kept 50 ms or more after the previous attack. |
| hit (event) | The span from one main attack (at most 25 dB under the file's loudest 5 ms) to the last 5 ms at or above its own line, before the next attack. Hits never overlap. |
| take | A run of hits. A new take starts when 200 ms or more under the sound line comes before an attack. |
| front | The first sample of a hit's attack, measured on the file itself (section 9.4). |
| natural end | Where a sound's tail settles to its floor (section 9.4). |
| null + reason | A measure that can't be taken honestly is `null`, and its `*Why` companion says why (section 11). Numbers are never faked. |

## 3. Package layout

An analysed library is a folder with one manifest and one JSON Lines file per layer. Each line holds one audio file's record. Layers join on `id`.

| File | Content | Required |
|---|---|---|
| `manifest.json` | format version, analyser name and version, decoder name and version, every layer's version and constants (section 4) | yes |
| `profile.jsonl` | layer P: facts, quality flags, the 74-measure shape of the whole file and of up to 16 hits (sections 6-8) | yes |
| `edit.jsonl` | layer E: every hit and take (up to 256 each) with fronts, peaks, natural ends and tails, plus a 10 ms level curve (section 9) | yes |
| `describe.jsonl` | layer D: space, distortion, resonances, stereo, loudness, pitch trend, background measures and the plain-words sound card (section 10) | yes from v0.1 |
| `names.jsonl` | layer N: UCS category and sub-category, FX name, keywords and description, joined by `id` (section 10.9) | optional |
| `vectors.*` | layer V: text and audio search vectors; the model name and version MUST be in the manifest | optional, off in v0.1 |
| `usage.jsonl` | layer U: the owner's own use (times used, ratings). PRIVATE: MUST NOT be shared or published | optional |
| `files.jsonl` | identity, size, date and measure versions per file, used to find renamed, moved or changed files | yes |

Every record line starts with:

| Field | Type | Meaning |
|---|---|---|
| `id` | string | audio identity: `<kind>:<len>:<24 hex>`. kind = `wav` / `aiff` / `flac` / `file`; len = byte length of the audio data chunk. The hash is the first 24 hex of MD5 over the kind, the length, the format chunk and sampled audio blocks (16 x 8 KB spread evenly, or all the data if it is 128 KB or less). A renamed or moved file keeps its `id`. |
| `rel` | string | the path RELATIVE to the analysed root. Absolute paths MUST NOT appear in a package that leaves the owner's machine. |
| `skip` | bool, only on skip lines | true when the file isn't readable audio. Then `reason` gives the decoder's error, `too short (under 20 ms)` or `silent file`, and nothing else follows. |

## 4. Versions

- `manifest.formatVersion`: semantic version of this specification (`0.1.0`).
- `manifest.measures`: one integer version per measure (below). It goes up whenever that measure's maths changes. A stored number is valid only for the measure version it was made under, recorded per file in `files.jsonl`. A tool reading the package MUST treat a mismatch as stale. An analyser SHOULD re-make stale or changed lines.
- `manifest.decoder`: the decoder's name, exact version, the downmix rule and the resampler settings. Required, because listening-view positions depend on them (open item O1).
- `manifest.constants`: every threshold, band edge and window the measures use, with its value, word for word as the analyser uses it. Header values MUST come from the constants that drive the maths, never from separate literals.

Measures and their v0.1 versions:
- P layer: `reader 1`, `events 1`, `envelope 1`, `length 1`, `colour 1`, `tone 1`, `texture 1`, `flags 1`.
- E layer: `eventlist 2`, `takes 1`, `fronts 3`, `tails 3`, `levels 1`, `kind 1`, `levelenv 1`.
- D layer: `space 1`, `dist 1`, `reson 1`, `stereo 1`, `loud 1`, `pitch 1`, `bg 1`, `card 1`.

## 5. Decoding

- Listening view (P and E layers): the first audio stream; the first 600 s; one channel; 32,000 Hz; float32. v0.1 records whatever downmix and resampler the decoder used (see O1).
- FRONT copy (E layer fronts and tails): the file read at its own sample format. The channel mean is the plain arithmetic mean of all channels, or a single named channel (`frontChannel`). It is resampled to 48,000 Hz with a windowed-sinc filter (Kaiser beta 5.0, half-length 10 x max(up, down) taps). A 48 kHz file is its own copy. Copy position v maps to the file sample `floor((2*v*down + up) / (2*up))`.
- Describe view (D layer): the file at its own rate, all channels, the first 600 s.
- Files accepted: WAV / RF64 / BW64, AIFF / AIFC and FLAC. In v0.1, FLAC files get no fronts or tails (O4).

## 6. Layer P record: `facts` (12) and `flags` (10)

`facts`:

| Field | Type | Unit | Meaning |
|---|---|---|---|
| durationSec | number or null | s | container duration (not limited to 600 s) |
| sampleRate | int or null | Hz | file rate |
| channels | int or null | count | from the container |
| codec | string | | decoder codec name, e.g. `pcm_s24le` |
| truncated | bool | | true when only the first 600 s were heard |
| heardSec | number | s | length of the listening view |
| peakDbfs | number | dBpk | largest sample of the listening view |
| levelDb | number | dBpow | loudest 5 ms level |
| leadSilenceMs | int | ms | first 5 ms at or above the sound line |
| activeMs | int | ms | total length of all hits (gaps between hits not counted) |
| activeDb | number | dBpow | mean level over the active ms |
| eventsFound | int | count | every hit found (uncapped) |

`flags` (quality filters; never used as distances):

| Field | Type | Unit | Meaning |
|---|---|---|---|
| lowFidelity | bool | | top end under 11 kHz, OR a noisy, compressed-sounding file (peak more than 15 dB over the median level AND floor within 35 dB of peak) |
| thin | bool | | 47-94 Hz is 30 dB or more under 200-797 Hz |
| artifacts | bool | | clipRuns >= 5, or any dropout, or hum found |
| notes | string[] | | the same findings in short English |
| topEndHz | int | Hz | highest frequency within 70 dB of the spectrum's maximum (0 = none) |
| floorRel | number | dB | floor minus loudest 5 ms |
| lowCliffDb | number | dB | mean level 47-94 Hz minus mean level 203-797 Hz |
| clipRuns | int | count | runs of 3+ samples at |x| >= 0.98 (on the listening view) |
| dropouts | int | count | runs of 64+ samples (2 ms) of digital zero inside hits |
| hum | int | Hz | 0, 50 or 60 |

## 7. Layer P record: shape, hits and main

| Field | Type | Meaning |
|---|---|---|
| `whole.v` | number[74] | the shape (section 8) of all hits joined end to end, gaps removed |
| `events` | array, 1-16 | the hits. If more than 16, the 16 loudest are kept, in time order |
| `events[].start` | int | the hit's start, in file samples: where its 1 ms power first comes within 20 dB of its own peak. Resolution 1/32000 s |
| `events[].start32` | int | the same, in listening-view samples |
| `events[].endMs` | int | the hit's end (exclusive) |
| `events[].peakRel` | number | the hit's loudest 5 ms minus the file's loudest 5 ms, in dB |
| `events[].gapAfterMs` | int or null | silence from this hit's end to the next attack; null for the last hit |
| `events[].v` | number[74] | the hit's own shape |
| `main` | int | index in `events` of the loudest hit |

## 8. The 74-measure shape

The spectral frames are 2048 samples (64 ms) every 10 ms, with a half-sample-shifted Hann window (`0.5 - 0.5*cos(2*pi*(i+0.5)/n)`), 15.625 Hz bins and DC excluded. A frame more than 70 dB under the loudest 5 ms is silent and skipped. The "joined line" is the shape's ranges laid end to end. All levels are relative to the shape's own peak and floored at -80 dB.

| Index | Names | Unit | Meaning |
|---|---|---|---|
| 0 | ttp | log10(1+ms) | time to the loudest 5 ms |
| 1 | rise | log10(1+ms) | time from -20 dB to -3 dB on the way to the peak |
| 2 | steep | dB per 5 ms (0-60) | the steepest 5 ms climb before the peak |
| 3-14 | at5, at10, at20, at40, at80, at160, at320, at640, at1280, at2560, at5120, at10240 | dB (-80 to 0) | the level that many ms after the START of the joined line |
| 15-22 | seg1-seg8 | dB | mean level of each eighth of the joined line |
| 23-24 | fall20, fall40 | log10(1+ms) | time from the peak to 20 dB and 40 dB down |
| 25 | hold | dB | median level of the middle half |
| 26 | attacks | log2(1+count) | number of hits (always 1 for a single hit) |
| 27 | length | log10(ms) | length of the joined line |
| 28-63 | att_*, early_*, late_*, whole_* with suffixes low, sub, b2, b3, b4, b5, b6, b7, bright | see below | colour at four moments: `att` = first 50 ms, `early` = 50-300 ms, `late` = 300 ms on, `whole` = all |
| 64 | flatness | log10 ratio (<= 0) | power-weighted spectral flatness, 109 Hz-12 kHz (noise-like toward 0) |
| 65 | periodicity | 0-1 | how periodic (tonal) |
| 66 | pitch | semitones re A4 = 440 Hz | power-weighted median pitch of periodic frames; read it with pitchConf (0 also means none) |
| 67 | pitchConf | 0-1 | share of power in clearly periodic frames |
| 68 | roughness | dB RMS | fine (1 ms) level movement against the 250 ms level |
| 69-71 | flutter, rough, grain | share 0-1 | share of that movement at 4-16 Hz, 16-64 Hz and 64-250 Hz |
| 72 | clicks | per second | clicks, at least 10 dB over their neighbourhood and 5 ms apart |
| 73 | jumpiness | 0-1 | mean spectral flux (how fast the spectrum changes) |

Colour suffixes. The bands are B0 0-100, B1 100-200, B2 200-400, B3 400-800, B4 800-1.6k, B5 1.6k-3.2k, B6 3.2k-6.4k and B7 6.4k-16k Hz.
- `low` = log10(share of all power under 200 Hz + 1e-4), from -4 to 0.
- `sub` = the same under 100 Hz.
- `b2`-`b7` = log10(the band's share of power at and above 200 Hz + 1e-3), from -3 to 0.
- `bright` = log2 of the power-weighted centroid of bins at and above 203 Hz, from about 7.67 to 13.97. 9.9658 (1 kHz) means there was no centroid.

## 9. Layer E (`edit.jsonl`)

### 9.1 Record fields

| Field | Type | Meaning |
|---|---|---|
| facts | object | as section 6 |
| kind | `oneshot` / `series` / `bed` | a label. `bed`: 3 takes or fewer, with (active share >= 0.7, crest <= 15 dB and a hit >= 2 s) OR (active share >= 0.9, crest <= 10 dB and not regular). `series`: 2+ takes or 2+ strong hits (within 6 dB of the loudest). Otherwise `oneshot` |
| activeShare | 0-1 | active ms / heard ms |
| crestDb | dB | levelDb - activeDb |
| spacingMs, spacingCv | ms, ratio, or null | the median start-to-start spacing of hits and its variation; null under 3 hits |
| regular | bool | spacingCv <= 0.35 |
| takesFound, takesKept, eventsKept | counts | |
| capped | bool | more than 256 hits found |
| floorDb, lineDb | dBpow | the floor and the sound line |
| frontReader | string | `ok`, or why fronts couldn't be measured: `no-audio`, `rate-differs`, `cannot-read:<code>` |
| frontChannel | int or null | the channel used for `first` fronts (0-based) |
| events | rows | 20 columns each (9.2) |
| takes | rows | 23 columns each (9.3) |
| envSteps, env10 | int, base64 | the level every 10 ms: the highest 5 ms level in each step, whole dB + 140, clamped 0-255, one byte per step, raw-deflate compressed, base64 |

### 9.2 Hit rows: column order

`start, endMs, peakRel, gapAfterMs, peakMs, peakDb, loudDb, front, frontBy, end, tailMs, frontFlag, tailFlag, frontRule, tailFloorDb, endLevelDb, minor, riseMs, secondHitMs, quietBeforeMs`

- `start`, `front`, `end` are file samples.
- `peakMs` is the ms of the hit's loudest 5 ms.
- `peakDb` is dBpk; `loudDb` and `endLevelDb` are dBpow; `tailFloorDb` is dBamp.
- `front` is the measured attack front; `frontBy` is `main` or `first`; `frontRule` is `floor` or `peak`.
- `end` is the natural end; `tailMs` runs from the peak to the end.
- `minor` = more than 18 dB under its take's loudest hit.
- `riseMs` runs from the front to the peak.
- `secondHitMs` = a second attack within 50 ms (a flam or double).
- `quietBeforeMs` = ms under the sound line just before this attack.
- Flags: `frontFlag` is one of `none-found`, `no-rise`, `slow-rise`, `before-its-start`, `after-its-peak` or `on-look-start`. `tailFlag` is one of `cut-by-next`, `no-floor`, `floor-from-sound`, `file-heard-in-part` or `runs-to-the-end`.

### 9.3 Take rows: column order

`event, events, start, endMs, peakRel, peakMs, peakDb, loudDb, front, peak, end, tailMs, frontFlag, tailFlag, frontRule, firstFront, firstFrontBy, firstFrontRule, firstFrontFlag, tailFloorDb, endLevelDb, riseMs, firstRiseMs`

`event` is the index of the take's first hit row (null if that hit wasn't kept). `peak` is the take's peak sample. The `first*` columns are the front of the take's first hit.

### 9.4 Fronts and natural ends

- `main` front: on the channel mean of the 48 kHz copy.
  - A 6 ms RMS envelope is looked at every 1 ms. The candidate is the look before the strongest rise of more than 1.5 dB, scored by the energy of the next 50 ms.
  - peak = the largest sample in the 50 ms after the candidate.
  - threshold = the higher of (floor + 12 dB, where the floor is the 10th percentile of the 1 ms RMS over the previous 120 ms) and (peak - 20 dB).
  - front = the first sample over the threshold, walking back from the peak while the envelope stays up, at most 120 ms.
- `first` front: for a hit under 300 ms after the previous one. It's measured per channel on 1 ms blocks: the first sample over -20 dB of the largest sample, looking back 20 blocks from the loudest block.
- A front is stored only if it lies between `start - 10 ms` and `peakMs + 10 ms`. Otherwise it's null, and `frontFlag` says why.
- Natural end:
  - A 20 ms RMS envelope runs over the region from the front to the stretch end (the quietest 5 ms in the 120 ms before the next attack).
  - tail floor = the median of the last 200 ms.
  - end = the last sample over floor + 6 dB.

## 10. Layer D (`describe.jsonl`), new in v0.1

Hit-based measures use the main hit's front and natural end from layer E, and record which E version they used. Each measure is null, with a reason, when it can't be taken honestly.

| Group | Fields | Unit | Meaning |
|---|---|---|---|
| 10.1 space | `decayS.low`, `decayS.mid`, `decayS.high`, with fit quality `decayR2.low/mid/high`; `earlyLateDb`; `bakedRoom` | s; dB; bool | the main hit's tail decay to -60 dB, extrapolated from a -5 to -25 dB Schroeder fit, in 3 bands (under 250 Hz, 250 Hz-2 kHz, over 2 kHz); the 0-50 ms vs later energy ratio; a room-in-the-file flag by a fixed rule. Null (`short-tail`) when the tail has under 20 dB of decay above the floor |
| 10.2 dist | `clippedSamples`, `saturation`, `crestDb` | count; 0-1; dB | samples at full scale in runs; a saturation score from flat tops and crest deficit; the crest factor behind it |
| 10.3 reson | `peaks[5]` of {hz, overFloorDb, decayMs}; `hnrDb` | Hz, dB, ms; dB | the five strongest resonances in the main hit's tail; harmonic-to-noise ratio |
| 10.4 stereo | `layout`, `widthDb`, `balanceDb`, `corr`, `corrWorst400ms`, `monoLossDb` | | channel layout; side over mid; L/R balance; phase correlation (whole file and worst 400 ms); level lost when folded to mono. Mono files carry `layout` only, and the rest is null (`mono`). Files with 3+ channels use the first two (front L/R) |
| 10.5 loud | `lufsI`, `lufsSMax`, `lra`, `truePeakDbtp` | LUFS, LUFS, LU, dBTP | BS.1770-4 / Tech 3342, all channels, file rate |
| 10.6 pitch | `trend`, `trendSemis`, `f0Hz` | `steady` / `rising` / `falling`; semitones; Hz | from the P layer pitch tracker, with the median pitch behind it. Null (`no-pitch`) when there's no stable pitch |
| 10.7 bg | `steadinessLu`, `eventsPerMin`, `loopSeam`, `loopSeamLevelDb`, `loopSeamSpecDb` | LU; count; 0-1; dB; dB | the spread of 1 s loudness; hits per minute; how well the last 0.5 s meets the first 0.5 s, with its level and spectrum parts |
| 10.8 card | `card` | text, 200 characters at most | plain words built from the numbers by a fixed, versioned rule table. No names, no AI. Null measures are left out. Example: "short sharp metallic hit, bright, ring near 2.1 kHz decaying 0.8 s, dry, mono" |
| 10.9 names (layer N, joined) | `ucsCatId`, `ucsCategory`, `ucsSubCategory`, `fxName`, `keywords`, `description`, `library`, `vendor` | text | what the sound IS. From the file's metadata or a UCS-style name, joined by `id` |

The exact definitions are fixed in the D layer's header blocks `constants`, `cardRules` and `measures`, quoted word for word by `manifest.constants` once the layer passes its known-answer tests.

## 11. Nulls and reasons

- A field is `null` when it was not measured. A `null` is never "no" or "zero".
- A nullable D-layer measure has a companion `<field>Why` ONLY when the field is null. Per-band measures carry a per-band object (e.g. `decaySWhy: {"high": "short-tail"}`); a resonance peak carries its own `decayMsWhy`. Reasons come from a fixed list:
  - `short-tail`: under 20 dB of decay above the floor
  - `no-pitch`: no stable pitch
  - `mono`: one channel
  - `fold-cancels`: left and right nearly cancel when folded to mono
  - `too-short`: under 400 ms for integrated loudness; under 3 s for short-term loudness, loudness range and steadiness
  - `too-short-for-seam`: too short for a loop seam
  - `no-edit-row`: no layer E row for the file, so hit-based measures can't be taken
  - `no-floor`
  - `not-measured`
  - `cannot-read:<code>`
- P and E keep their existing flag columns as their reasons.

## 12. Privacy and publishing

- Public: this specification, the analyser and its known-answer test set.
- Never public: anyone's index data. That covers paths, folder walks, identities, sizes, dates, library statistics (`scales`) and the usage layer.
- A package that leaves the owner's machine MUST use `rel` paths only, and MUST drop `files.jsonl`'s sizes and dates, the run timings (`took`, `tookEdit`) and the logs.
- Run timings are not sound measures. They live in a separate optional `run.jsonl`, never in records.

## 13. Conformance

- An analyser conforms when it passes the public known-answer set: synthetic files with known answers for every measure. Examples: set decay times per band, a clipped sine, damped tones, set stereo width and correlation, the EBU 3341 loudness signals, a pitch glide, a perfect loop. It also has to reproduce the reference outputs of a small free sample library, byte for byte, apart from run data.
- Two runs of the same analyser version on the same file MUST give identical records.

## 14. Open items before 1.0

- O1. The listening-view downmix and resampler are the decoder's defaults and no decoder version is recorded, so positions can differ slightly between machines. Plan: pin a plain-mean downmix, a named resampler with fixed settings and the decoder version in the manifest (`reader 2`).
- O2. Pitch and periodicity use a circular per-frame autocorrelation normalised by a linear window autocorrelation. Plan: keep for v0.1, document, review in `tone 2`.
- O3. `minor` and `secondHitMs` are null when fronts can't be read, although they don't need the file. Plan: always compute (`eventlist 3`).
- O4. FLAC files get no fronts or tails. Plan: read FLAC for the FRONT copy (`fronts 4`).
- O5. An analyser crash writes no line. Plan: write a skip line with reason `analyser-error`.
- O6. Stale or changed files are reported but not re-made. Plan: a re-make pass.
- O7. The field-to-measure map lives only in code comments. Plan: put it in the manifest.
- O8. Plan: record the analyser build hash in the manifest.
- O9. Run timings move to `run.jsonl` (section 12).

## 15. Licence

This specification is released under the MIT License (see LICENSE), like ucs-tools. It complements the Universal Category System (UCS): UCS names what a sound is; this format measures how it sounds and where its hits are.
