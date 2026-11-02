# Designing questions

## The four question types

Every question has an ID you choose (it comes back as the key of its answer and isn't sent to the model), a `type` and `instructions` (the question itself).

### `noul`: yes or no

```json
"frustrated": {
  "type": "noul",
  "instructions": "Does the caller sound frustrated?",
  "criteria": {
    "true": "Frustration is audible in the caller's voice",
    "false": "The caller sounds calm, or only says in words that they're frustrated"
  }
}
```

The answer is the share of raters who'd say yes: `{ "type": "noul", "noul": 0.81 }`.

Use it for detection: is something there or not. `criteria` is optional: use it to say what counts as yes and what counts as no when the question alone could be read more than one way.

### `choice`: which one of several

```json
"tone": {
  "type": "choice",
  "instructions": "Which best describes the agent's tone?",
  "criteria": [
    { "name": "warm", "description": "Friendly and caring toward the caller" },
    { "name": "neutral", "description": "Polite but impersonal" },
    { "name": "curt", "description": "Short or dismissive" }
  ]
}
```

The answer has the likeliest option, a `confidence`, and a probability per option name. Use it for classification. Names must be unique, and you need 2 to 100 options.

### `score`: how much, on an ordered scale

```json
"patience": {
  "type": "score",
  "instructions": "How patient does the agent sound?",
  "criteria": ["Not at all", "Slightly", "Moderately", "Very", "Extremely"]
}
```

Levels are listed lowest first and numbered from 0. The answer has `score` (the mean level, so 2.6 means between "Moderately" and "Very"), a `confidence`, and a probability and label per level. Use it for ranking and intensity.

### `moment`: when something happens

```json
"laughs": {
  "type": "moment",
  "instructions": "When does the caller laugh?"
}
```

The answer is a list of `moments`: time ranges in milliseconds from the start of the file, each with a probability. An empty list means the event was not found. A `moment` answer gives times but no share of raters, so to ask whether an event occurs as well as when, add a `noul` question. Treat moments as candidates and confirm them (see [audio-units.md](audio-units.md#where-or-how-often-something-happens)).

## Closed choices work best

The API is strongest when you supply the possible answers. "Which of these languages is spoken: English, Spanish, French, Japanese, none of these?" works far better than hoping it can name any language. The same holds for accents, intents, request types, sound types and candidate words.

- List the options your application actually needs to tell apart.
- Make them mutually exclusive.
- Add "None of these" or "Unclear" when many clips won't fit any option, so they aren't forced into a wrong one.
- Give each option a short name, plus a `description` when the name alone could be read two ways.

## Wording

- **One property per question.** "Does the agent sound warm and professional?" is two questions. Split it.
- **Name the person.** "the caller", "the agent", "the speaker on the left channel". With several voices in the clip, an unnamed question gets an answer about whoever is most prominent.
- **Keep it neutral.** Ask "How patient does the agent sound?", not "Isn't the agent impatient?".
- **Ask about what's audible.** "Does the caller sound frustrated?" is a listening question. "Is the caller frustrated?" asks about a private feeling. "Did the caller say they were frustrated?" is a question about the words: use the transcript.
- **Keep the same questions in the same order** for every recording you'll compare. Questions in a request are answered together, so adding, removing or reordering them can shift the answers.

## Scales

Short scales work best: 3 to 5 levels. Common ones that fit many questions:

| Scale | Levels |
| --- | --- |
| Intensity | Not at all / Slightly / Moderately / Very / Extremely |
| Likelihood | Definitely not / Probably not / Unsure / Probably / Definitely |
| Quality | Bad / Poor / Fair / Good / Excellent |
| Agreement | Strongly disagree / Disagree / Neutral / Agree / Strongly agree |
| Speed | Slow / Moderate / Fast |

Label every level with words. A bare 1–5 scale leaves raters to guess what each number means.

## Context

`context` is text the model reads alongside the audio. Use it for what a rater would need to know to answer well:

- **The setting:** "Customer support call for a mobile carrier. The agent speaks first."
- **Who is speaking:** "The clip is the caller's channel only."
- **The words being judged:** "The speaker is reading: 'Technology has changed how we view art today.'" This lets you ask which word is stressed, or whether a line was mispronounced.
- **Earlier turns:** "Earlier, the caller asked when their refund would arrive." This lets you ask whether the reply in the clip addresses it.
- **A list of candidates:** a tool catalog, a list of contact reasons, the expected script.

Don't put the question in `context`, and keep it short: context is read with every question in the request.

## Gates and follow-ups

The API answers every question you send in a request. To save work, chain requests in your code:

1. Ask one cheap gating question on everything ("Is anyone speaking?", "Does the caller sound upset at all?").
2. Ask the detailed questions only on the units where the gate says yes.

## Example questions by area

These follow the patterns above. Check each one on your own audio before relying on it. [capabilities.md](capabilities.md) says which areas are well supported and which work only under conditions.

| Area | Question | Type | Options or scale |
| --- | --- | --- | --- |
| Language | Which language is spoken? | `choice` | the languages you expect, plus "None of these" |
| Language | Does the speaker switch between English and Spanish? | `noul` | |
| Speech | Is the speaking rate slow, moderate or fast? | `score` | Slow / Moderate / Fast |
| Speech | Is there a filled pause ("uh", "um") in this clip? | `noul` | |
| Voice | Which accent best describes the speaker? | `choice` | the accents you need to tell apart, plus "None of these" |
| Voice | Is this speech artificially generated? | `noul` | |
| Emotion | Which emotion does the speaker convey in their voice? | `choice` | e.g. happy / sad / angry / fearful / neutral |
| Emotion | Which word carries the emphasis? (sentence in `context`) | `choice` | candidate words from the sentence |
| Environment | Is there noise behind the speaker? | `noul` | |
| Environment | What kind of sound is under the voice? | `choice` | Hiss / Hum / Other people talking / Music / None |
| Quality | How would most listeners rate the sound quality of this speech? | `score` | Bad / Poor / Fair / Good / Excellent |
| Quality | How natural does this voice sound? | `score` | Bad / Poor / Fair / Good / Excellent |
| Non-speech | Does anyone laugh in this clip? | `noul` | |
| Non-speech | What type of non-verbal sound is in the clip? | `choice` | Cough / Breath / Cry / Laugh / Sigh / None |
| Conversation | Do two speakers talk at the same time? | `noul` | |
| Conversation | Someone speaks during the main speaker's turn. Do they take over, or does the main speaker carry on? | `choice` | Takes over / Brief acknowledgment |
| Intent | What is the caller asking for? | `choice` | your contact reasons, plus "Something else" |
| Intent | Did the reply address what the caller asked? (earlier turn in `context`) | `score` | Not at all … Directly addresses it |
