# Example: reviewing many recordings

**Goal:** score a folder of support-call recordings, some longer than 5 minutes, with the same questions, and export one row per call.

**Units:** the caller's channel, in chunks of up to 5 minutes cut by the SDK's `create_chunked`. Answers are combined per call: the mean for overall levels, the maximum for "did it ever happen".

**Questions:** three about the caller, the same set in the same order for every chunk.

```python
"""Score every recording in a folder and write one CSV row per call.

Needs: pip install hume-audio-decisions; ffmpeg on the PATH; HUME_API_KEY set.
"""
import csv
import subprocess
import sys
import tempfile
from concurrent.futures import ThreadPoolExecutor
from pathlib import Path

from hume_audio_decisions import (
    Answer_Noul, Answer_Score, DecisionsClient, DecisionsRequest, Question_Noul, Question_Score,
)

CHUNK_MS = 300_000           # the longest chunk the API accepts
CALLER_CHANNEL = 0           # check which channel is the caller on a few files first
PARALLEL = 5                 # calls processed at once; stay within your rate limit

# Retries 408, 409, 429 and 5xx responses, waiting as long as Retry-After asks.
client = DecisionsClient(max_retries=5)

REQUEST = DecisionsRequest(
    # Holds for every chunk: the model answers each chunk as if it were the whole call.
    context="Customer support call. This audio is the caller's side only.",
    questions={
        "frustration": Question_Score(
            instructions="How frustrated does the caller sound?",
            criteria=["Not at all", "Slightly", "Moderately", "Very", "Extremely"],
        ),
        "raised_voice": Question_Noul(instructions="Does the caller raise their voice?"),
        "noise": Question_Noul(instructions="Is there distracting noise behind the caller?"),
    },
)


def caller_track(src: Path, out_dir: Path) -> Path:
    """Take the caller's channel as 16 kHz mono WAV."""
    out = out_dir / f"{src.stem}.caller.wav"
    subprocess.run(
        ["ffmpeg", "-loglevel", "error", "-y", "-i", str(src),
         "-af", f"pan=mono|c0=c{CALLER_CHANNEL}", "-ar", "16000", str(out)],
        check=True,
    )
    return out


def score_call(src: Path) -> dict:
    with tempfile.TemporaryDirectory() as tmp:
        track = caller_track(src, Path(tmp))
        with open(track, "rb") as audio:
            # Cuts the call into equal chunks of up to CHUNK_MS and sends them one at a time.
            result = client.decisions.create_chunked(
                file=(track.name, audio, "audio/wav"), request=REQUEST, max_chunk_ms=CHUNK_MS
            )

    # A failed chunk doesn't raise: its error is recorded on the chunk.
    if not result.complete:
        failed = [f"chunk {c.index}: {c.status}, {c.error}" for c in result.chunks if c.response is None]
        raise RuntimeError(f"{src.name}: {failed}")

    done = [c for c in result.chunks if c.response is not None]
    seconds = [(c.end_ms - c.start_ms) / 1000 for c in done]
    answers = [c.response.answers for c in done]
    total = sum(seconds)

    frustration = [a["frustration"].score for a in answers if isinstance(a["frustration"], Answer_Score)]
    raised = [a["raised_voice"].noul for a in answers if isinstance(a["raised_voice"], Answer_Noul)]
    noise = [a["noise"].noul for a in answers if isinstance(a["noise"], Answer_Noul)]
    return {
        "file": src.name,
        "minutes": round(total / 60, 1),
        "chunks": len(done),
        # Overall level: mean weighted by chunk length.
        "frustration_mean": round(sum(f * s for f, s in zip(frustration, seconds)) / total, 2),
        # "Did it ever happen": the highest chunk.
        "frustration_max": round(max(frustration), 2),
        "raised_voice_max": round(max(raised), 3),
        "noise_mean": round(sum(noise) / len(noise), 3),
    }


if __name__ == "__main__":
    files = sorted(p for p in Path(sys.argv[1]).iterdir() if p.suffix.lower() in {".wav", ".mp3"})
    with ThreadPoolExecutor(PARALLEL) as pool:
        rows = list(pool.map(score_call, files))
    with open("results.csv", "w", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=rows[0].keys())
        writer.writeheader()
        writer.writerows(rows)
    print(f"Wrote {len(rows)} rows to results.csv")
```

Every file must be at least 5 seconds long: the API refuses shorter audio with `audio_too_short`.

## What to do next

- **Pick thresholds on labelled calls.** Have someone mark 50 calls "needs review" or not, then choose the `frustration_max` and `raised_voice_max` cut-offs that separate them best. Don't start from 0.5.
- **Send the middle band to people.** Calls whose answers sit between two thresholds go to a reviewer instead of being decided automatically.
- **Listen** to the five highest and five lowest calls on each column before trusting it.
- **Store the raw answers** (per chunk, with the `run_id`) as well as the summary, so you can recombine them later without re-running. `iter_create_chunked` yields each chunk's response as soon as it arrives, so you can save it before the next chunk is sent.
- **Estimate the cost first.** `client.decisions.input_tokens` counts the input tokens a chunk would be billed for, from its duration and the questions, without sending audio.
