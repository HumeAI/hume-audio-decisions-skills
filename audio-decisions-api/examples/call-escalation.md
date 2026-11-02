# Example: is this support call getting heated?

**Goal:** flag support calls a supervisor should listen to. This is an outcome question, not a single listening question, so it's built from parts ([../references/breaking-down-questions.md](../references/breaking-down-questions.md#example-is-this-support-call-getting-heated)). Treat the combined flag as unproven until you've checked it against supervisors' own flags.

**Inputs:** a stereo recording (caller on one channel, agent on the other) and a transcript with speaker turns and timestamps from your speech-to-text provider.

**Units:**
- Each caller turn of at least 2 s, from the caller's channel, padded with the audio around it to the API's 5 s minimum: how the caller sounds.
- A 5 s window of the mixed audio centred on each point where the agent starts talking while the caller is still talking: does the agent take over?

**Other signals:** the transcript, for insults, threats and requests for a manager.

```python
"""Flag calls that look heated. Needs: hume-audio-decisions, ffmpeg, HUME_API_KEY."""
import re
import subprocess
from dataclasses import dataclass
from pathlib import Path

from hume_audio_decisions import (
    Answer_Choice, Answer_Noul, Answer_Score, Criterion, DecisionsClient, DecisionsRequest,
    Question_Choice, Question_Noul, Question_Score,
)

client = DecisionsClient()


@dataclass
class Turn:
    speaker: str   # "caller" or "agent"
    start: float   # seconds
    end: float
    text: str


CALLER_REQUEST = DecisionsRequest(
    context="Customer support call. This clip is the caller's side only, centred on one of their turns.",
    questions={
        "frustration": Question_Score(
            instructions="How frustrated does the caller sound?",
            criteria=["Not at all", "Slightly", "Moderately", "Very", "Extremely"],
        ),
        "raised_voice": Question_Noul(instructions="Does the caller raise their voice?"),
    },
)

OVERLAP_REQUEST = DecisionsRequest(
    context="Customer support call, both sides mixed. The agent starts talking while the caller is speaking.",
    questions={
        "takeover": Question_Choice(
            instructions="Does the agent take over the turn, or briefly acknowledge while the caller carries on?",
            criteria=[
                Criterion(name="takes_over", description="The agent's speech takes the turn; the caller stops"),
                Criterion(name="acknowledges", description="A brief 'mm-hm' or 'okay'; the caller keeps talking"),
            ],
        ),
    },
)

# Transcript signals. Use your own lists, or an LLM for anything subtler.
INSULTS = re.compile(r"\b(idiot|useless|stupid|ridiculous)\b", re.I)
ESCALATION = re.compile(r"\b(manager|supervisor|cancel|lawyer)\b", re.I)

MIN_CLIP_S = 5.0  # the API refuses shorter audio with audio_too_short


def window(start: float, end: float) -> tuple[float, float]:
    """Centre a clip of at least MIN_CLIP_S on start..end. Returns (clip start, clip seconds)."""
    seconds = max(end - start, MIN_CLIP_S)
    return max(0.0, (start + end - seconds) / 2), seconds


def cut(src: Path, start: float, seconds: float, out: Path, channel: int | None) -> Path:
    pan = ["-af", f"pan=mono|c0=c{channel}"] if channel is not None else ["-ac", "1"]
    subprocess.run(
        ["ffmpeg", "-loglevel", "error", "-y", "-ss", f"{start:.2f}", "-t", f"{seconds:.2f}",
         "-i", str(src), *pan, "-ar", "16000", str(out)],
        check=True,
    )
    return out


def ask(path: Path, request: DecisionsRequest):
    with open(path, "rb") as audio:
        return client.decisions.create(file=(path.name, audio, "audio/wav"), request=request).answers


def review_call(call: Path, turns: list[Turn], tmp: Path, caller_channel: int = 0) -> dict:
    # 1. How the caller sounds, turn by turn.
    caller = []
    for i, t in enumerate(x for x in turns if x.speaker == "caller" and x.end - x.start >= 2):
        start, seconds = window(t.start, t.end)
        a = ask(cut(call, start, seconds, tmp / f"c{i}.wav", caller_channel), CALLER_REQUEST)
        f, r = a["frustration"], a["raised_voice"]
        caller.append({
            "turn": t,
            "frustration": f.score if isinstance(f, Answer_Score) else 0.0,
            "raised": r.noul if isinstance(r, Answer_Noul) else 0.0,
            "insult": bool(INSULTS.search(t.text)),
        })

    # 2. Does the agent take over when they talk over the caller?
    takeovers = 0
    for i, agent in enumerate(t for t in turns if t.speaker == "agent"):
        talking_over = any(c.speaker == "caller" and c.start < agent.start < c.end for c in turns)
        if not talking_over:
            continue
        start, seconds = window(agent.start, agent.start)
        a = ask(cut(call, start, seconds, tmp / f"o{i}.wav", None), OVERLAP_REQUEST)["takeover"]
        if isinstance(a, Answer_Choice) and a.probabilities["takes_over"] > 0.6:  # tune on your calls
            takeovers += 1

    # 3. Combine. Thresholds here are starting points: tune them on calls supervisors have flagged.
    VERY = 3.0  # "Very" on the frustration scale
    two_in_a_row = any(
        a["frustration"] >= VERY and b["frustration"] >= VERY for a, b in zip(caller, caller[1:])
    )
    loud_and_insulting = any(c["raised"] > 0.7 and c["insult"] for c in caller)
    agent_overriding_upset_caller = takeovers >= 2 and any(c["frustration"] >= VERY - 1 for c in caller)
    asked_for_escalation = any(ESCALATION.search(t.text) for t in turns if t.speaker == "caller")

    reasons = [name for name, hit in [
        ("caller frustrated two turns running", two_in_a_row),
        ("raised voice with an insult", loud_and_insulting),
        ("agent talked over an upset caller", agent_overriding_upset_caller),
        ("asked for a manager or to cancel", asked_for_escalation),
    ] if hit]
    return {"call": call.name, "flag": bool(reasons), "reasons": reasons, "turns_scored": len(caller)}
```

## Checking it

1. Collect 100 past calls with supervisors' "should have been escalated" flags. Use half to tune the thresholds and the other half to test.
2. Check each listening part first: listen to the turns with the highest and lowest frustration, and the overlaps judged "takes over".
3. Measure the flag on the test half: how many flagged calls did supervisors agree with, and how many of their flags did it catch?
4. Remove the listening parts (keep only the transcript rules) and measure again. The difference is what the audio adds.
5. Report the result as checked on your calls, with the thresholds you chose. Don't call it an aggression or escalation detector beyond that.
