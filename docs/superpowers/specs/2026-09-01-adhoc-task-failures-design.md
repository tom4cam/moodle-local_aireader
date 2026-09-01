# Spec: Stop the deterministic adhoc task failures

**Date:** 2026-09-01
**Status:** Approved (brainstorming)

## Goal

Clear the two permanently failing adhoc task types on learn.saylor.org (26
`align_audio`, 7 `generate_audio`, all retrying daily at the capped 86400s
delay) and make both failure modes self-correcting rather than fatal.

## The two failures, as measured

**`generate_audio`**, at `classes/task/generate_audio.php:149`:

```
TTS request failed: HTTP 400: Input of 2159 tokens is over the maximum
input limit of 2000 tokens.
```

**`align_audio`**, at `classes/task/align_audio.php:98`:

```
aligning asset 4871 (35925942 bytes) via whisper-1
Whisper alignment request failed: HTTP 413: Maximum content size limit
(26214400) exceeded (26326537 bytes read)
```

## Root cause of the TTS failure, corrected

The original report proposed a more conservative characters-per-token ratio.
That would not have worked, and the reason matters for the design.

`openai_client.php:44` documents the author's intent: `gpt-4o-mini-tts` rejects
input over 2000 tokens, `tts-1`/`tts-1-hd` cap at 4096 characters, and the code
"applies both." The 3800-character cap is correctly sized for `tts-1`. The
token ceiling is the one that fails, because it is enforced through
`estimate_tokens()`, which counts non-CJK text as `length / 4`:

| | chars | estimated tokens | ceiling | binds? |
| --- | --- | --- | --- | --- |
| Current | 3800 | 950 | 1800 | no |
| Report's proposal (÷3, cap 1500) | 3800 | 1267 | 1500 | no |
| Observed reality | 3800 | **2159 actual** | 2000 (API) | rejected |

Reaching an 1800-token estimate takes 7200+ characters, which the 3800-character
cap makes unreachable. So the token ceiling is dead code for Latin text, and
the report's proposed ratio leaves it dead. The content that broke it ran at
roughly 1.8 characters per token, and translated text and exotic scripts go
lower still.

**Therefore the estimator is not the fix and is left untouched.** Any static
ratio is a prediction about content not yet seen. The API's verdict is the only
authority, so the design reacts to it.

## Decisions taken during brainstorming

- Treat the API's 400 as the authority: catch it and re-split, rather than
  tuning a ratio. Rejected: changing the estimator's divisor, which does not
  bind (see table) and only moves the guess.
- Leave `estimate_tokens()` alone entirely. Reliability comes from the
  character cap plus recursion depth, argued below, so the estimator does not
  enter the correctness argument. It stays as the CJK safeguard it already is.
- Make the character cap model-aware rather than clamping every site. 3800 is
  correct for `tts-1`/`tts-1-hd`; only `gpt-4o-mini-tts` needs 2400. This keeps
  the change acceptable upstream instead of imposing ~60% more TTS requests on
  sites that do not have the problem.
- For oversized audio, split and preserve karaoke rather than skipping it, and
  do it with a general MP3 splitter so the 26 already-failed assets heal
  without regeneration. Rejected: ffmpeg re-encoding (Moodle does not ship
  ffmpeg and its presence on learn.saylor.org is unverified, so it trades a
  deterministic API failure for a deterministic missing-binary failure);
  slicing at recorded chunk offsets (elegant and parser-free, but only helps
  newly generated assets).
- Deterministic failures must stop retrying. This is a separate fix from the
  other two and is the direct cause of the queue accumulation.

## Why this is reliable without the estimator

Worst-case reasoning, not calibration:

- A `gpt-4o-mini-tts` chunk enters at most 2400 characters.
- The retry splits on failure, up to 4 levels, so the smallest piece is about
  150 characters.
- At a pathological 4 tokens per character (what exotic scripts and dense
  punctuation actually reach), 150 characters is ~600 tokens against a 2000
  limit.

So depth 4 tolerates roughly 13 tokens per character at the leaf. No ratio
assumption is load-bearing.

## New and changed components

### `classes/manager/mp3_splitter.php` (new)

Pure PHP, no external binary. Follows `id3_writer.php`, which is the existing
precedent for byte-level MP3 work.

`split(string $mp3, int $maxbytes): array`
Returns a list of `['bytes' => string, 'duration' => float]`, each part at most
`$maxbytes`, split only on frame boundaries.

Implementation notes an implementer must not skip:

- **Skip a leading ID3v2 tag** using its synchsafe size (the same encoding
  `id3_writer::synchsafe()` writes). `generate_audio` prepends one via
  `id3_writer::tag_mp3()`, so it is always present on stored files.
- **Strip an ID3v1 trailer** (128 bytes beginning `TAG` at end of file) so it
  is not counted as frame data.
- **Read bitrate and sample rate per frame**, not once. VBR is real and a
  single read would desynchronise the walk.
- **Frame length** = `floor(144000 * bitrate_kbps / sample_rate) + padding` for
  MPEG1 Layer III; MPEG2/2.5 Layer III use 72000 rather than 144000.
- **Frame duration** = samples-per-frame / sample rate (1152 for MPEG1
  Layer III, 576 for MPEG2/2.5), which is what the caller needs for timestamp
  offsets.
- **Treat a Xing/Info frame as an ordinary frame.** It lives in the first frame
  and needs no special handling for this purpose.
- Each part is emitted as raw frames with no ID3 tag, which is a valid MP3
  stream and what Whisper accepts.

Returns an empty list when no valid frame is found, which the caller treats as
"cannot split" rather than an error.

### `classes/manager/segment_stitcher.php` (new)

`stitch(array $parts): array`
Pure array math, no I/O. Takes per-part segment lists paired with part
durations, offsets each part's `start` and `end` by the cumulative duration of
preceding parts, renumbers `id` sequentially, and returns one flat list in the
shape `segment_manager::store_for_asset()` already consumes.

### `classes/manager/openai_client.php` (extend)

- Add `MAX_CHUNK_SIZE_TOKEN_MODELS = 2400` and a model-to-cap resolver
  `chunk_size_for(string $model, int $configured): int`, returning
  `min($configured, 2400)` for `gpt-4o-mini-tts` and `min($configured, 3800)`
  otherwise. Pure and unit-testable.
- `synthesize()` throws a typed exception on the input-limit rejection so
  callers can branch on it. The status is currently folded into a sanitized
  string by `http_guard::sanitize_error()`, which preserves OpenAI's
  `error.message` but only as prose, so matching on text alone would be
  fragile.
- `DEFAULT_CHUNK_SIZE` and `estimate_tokens()` are unchanged.

### `classes/exception/tts_input_too_long.php` (new)

Extends `\moodle_exception`, carries the HTTP status and the API message.
Thrown only when the status is 400 **and** the message matches OpenAI's
input-limit wording. Every other 400 stays fatal, because a generic 400 means a
malformed request and re-splitting it would only burn calls.

### `classes/manager/tts_splitter.php` (new)

`synthesize_split(callable $synth, string $text, int $maxchars, int $depth = 0): string`

Calls `$synth($piece)`; on `tts_input_too_long`, re-chunks that piece with the
existing sentence-aware `openai_client::chunk_text()` at half the current cap
and recurses, concatenating the returned audio. Guards: maximum depth 4, and a
minimum piece length below which it rethrows, so a genuinely broken request
cannot spin.

Taking the synthesizer as a callable makes this fully unit-testable with no
curl seam, and callable injection is already the established pattern here:
`generate_audio` passes a closure to `translation_manager::get_or_translate()`.

### `classes/task/generate_audio.php` (extend)

Resolve the chunk cap through `chunk_size_for($asset->model, $configured)`, and
route each chunk through `synthesize_split()` with a closure wrapping
`$client->synthesize($chunk, $asset->model, $asset->voice, $instructions)`.

### `classes/task/align_audio.php` (extend)

When the stored file exceeds the Whisper ceiling, split it, align each part,
stitch the timestamps, and store the merged result. `openai_aligner::align()`
stays "align one file"; orchestration lives in the task, matching the existing
separation.

**Target 24 MiB (25165824) per part, not 25.** The failure shows the server
aborting mid-read at 26326537 bytes against a 26214400 limit, so the multipart
envelope counts against the cap. Headroom avoids a fix that fails at the
boundary.

If `mp3_splitter::split()` returns nothing, log and skip alignment cleanly.
A missing alignment already degrades gracefully: no segments means no karaoke
and the audio still plays.

## Retry semantics

Both tasks classify their failures instead of rethrowing unconditionally, which
`align_audio.php:101` does today and which is why these retry forever:

- Network errors, 5xx and 429: rethrow, so Moodle retries with backoff.
- Deterministic 4xx, or audio that cannot be parsed: record on the asset
  (`lasterror`, which already exists) and return normally, so the task
  completes and leaves the queue.

## Draining the existing 33 failures

Moodle retains failed adhoc tasks and retries them at the capped delay, so once
this deploys they should drain on their next retry with no migration or requeue
script. Two things to confirm during implementation rather than assume:

- whether the 26 `align_audio` failures are 26 distinct assets or duplicates of
  fewer, since duplicates would each pay for a redundant Whisper pass;
- whether `segment_manager::store_for_asset()` is idempotent, for the same
  reason.

## Testing

All offline, no network, matching the existing suite's conventions.

- `mp3_splitter`: synthetic MP3 fixtures built in the test from valid frame
  headers, so no binary blob is committed. Assert exact frame boundaries, VBR
  handling, ID3v2 skip, ID3v1 strip, that no part exceeds the cap, that
  durations sum to the whole, and that an unparseable body yields an empty list.
- `segment_stitcher`: offsets accumulate across parts, ids renumber, empty
  parts are tolerated.
- `tts_splitter`: a fake synthesizer closure that throws `tts_input_too_long`
  above a threshold; assert it splits, concatenates in order, respects max
  depth, and rethrows below the minimum piece size. Also assert a non-matching
  400 is not retried.
- `openai_client::chunk_size_for`: caps per model, honours a smaller configured
  value, ignores a larger one.
- `estimate_tokens()` keeps its existing tests unchanged, as evidence the
  estimator was not touched.

`synthesize()` and `openai_aligner::align()` remain the only network-touching
functions and stay untested, as they are today.

## Out of scope

- ffmpeg re-encoding.
- Changing `estimate_tokens()`.
- Any change to how the player consumes segments.
- Commits to the upstream `dta121` repo. This lands on a branch in the
  `tom4cam` fork; whether it goes upstream as a PR is a separate decision.
