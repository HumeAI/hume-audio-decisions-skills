# What the API answers well

This page sorts kinds of questions into groups by how well they're supported. It's guidance for choosing questions, not a guarantee: every application should check its own questions on its own audio ([SKILL.md, Step 4](../SKILL.md#step-4-check-before-relying-on-it)). The model can be updated, so re-check after updates.

## The test for any question

Could a group of listeners answer it the same way, from listening to this clip, without having to reason it through? If yes, it's a candidate. If it needs open-ended output, counting, outside knowledge or several steps of reasoning, break it down first ([breaking-down-questions.md](breaking-down-questions.md)).

## Generally works

These work well when asked as written, especially as closed choices with options you supply. Still choose your thresholds on your own clips.

| Area | Examples |
| --- | --- |
| Language | Whether speech switches between two named languages |
| Words from a list | Which of several similar-sounding words was said; which of a list of prompted words was spoken |
| Accent | Which accent best fits a speaker, from options you supply |
| Synthetic speech | Whether an utterance contains synthesized speech |
| Emphasis | Which word in a supplied sentence carries the stress |
| Intent | Which intent or request type a spoken request belongs to, from your list (for example banking requests, voice-assistant commands) |
| Speech acts | Whether an utterance is a question, a statement, a request and so on |
| Answers from supplied material | Which supplied answer matches what was asked; when in the clip a supplied answer or name is spoken (`span`) |
| Sound quality | How raters would rate the technical quality of transmitted speech |
| Non-speech | Whether someone laughs; which type of non-verbal sound it is (cough, breath, cry and so on) from options you supply |

## Works under conditions

These are supported, but each depends on conditions: short clips, supplied options or context, or audio like what the model learned from. Check them on your own audio before relying on them.

| Area | Examples | Conditions to watch |
| --- | --- | --- |
| Emotion | The emotion a voice conveys; whether a named emotion is present; sentiment of a spoken opinion | Strongest on short clips of one speaker. Weaker in long, natural conversations. Ask what the voice conveys, not what the person feels. |
| Delivery | Speaking rate (slow / moderate / fast), pitch, loudness changes, breathiness, monotone vs lively | Coarse levels, not measurements. Not "words per minute". |
| Disfluency | Filled pauses ("um"), repeated words or sounds | Windows of at least 10 s. Whether a filler is present, not why. |
| Voice | Synthetic origin of a whole clip, perceived human-likeness, naturalness | Depends on the voice generator and recording chain. New generators and heavy compression are uncertain. |
| Perceived traits | Perceived age, perceived sex, voice descriptors | Raters' perceptions, never facts about a person (see "Don't use"). |
| Environment | Background noise and its type, the acoustic setting, named sounds (a dog, an alarm) from your list, how intrusive the noise is | Closed lists of sound types. Music under speech is weaker. |
| Conversation | Whether two people talk over each other; whether someone interrupts or just acknowledges; pauses within a turn; the gap before a reply | Work on a selected excerpt around the moment, not across a whole call. |
| Timing | When something happens (`span`); where a pause falls relative to supplied words | Spans are candidates, not exact timestamps: confirm each with a `noul` question on a window around it. |
| Responses | Whether a reply is relevant to what was asked; whether it fits emotionally; whether a performance fits a line | Supply the earlier turn or the intended line in `context`. |
| Intelligibility and clarity | How much of the speech a listener can understand; clarity of a read-out | Phone audio and accents not yet well covered. |

## Unproven: build it from parts and check it

No direct evidence yet, but these can often be built from signals that are supported, combined with a transcript, other models and your own rules. Treat the combination as unproven until you've checked it against your own labelled examples. [breaking-down-questions.md](breaking-down-questions.md) has worked examples.

| Goal | Parts to build from |
| --- | --- |
| Aggression or hostility | Conveyed anger or frustration, intensity, interruptions that take over the turn, overlapping speech; the transcript for threats or insults |
| Bullying or harassment | The same tone and interruption signals per speaker across turns; speaker separation and the transcript for who said what to whom |
| Profanity | Speech-to-text and a word list for the words; the API for how they were said (shouted or casual) |
| Sarcasm or irony | Tone that conflicts with the words (words in `context`), plus an LLM reading the transcript |
| Empathy, helpfulness, call quality | Tone and pace questions about the agent, response relevance with the earlier turn in `context`, plus an LLM judge on the transcript |
| Business outcomes (escalation, churn, sale) | Listening questions as features, combined with call metadata and transcripts, checked against real outcomes |

## Use other tools

| Need | Use |
| --- | --- |
| Counting speakers or turns | Speaker separation (diarization) |
| Exact timestamps | Voice activity detection, forced alignment, or the confirm-with-`noul` pattern for rough times |
| Transcripts, summaries, open topic labels | Speech-to-text and an LLM |
| Open-world language identification | A language-ID model, or a `choice` over the languages you expect |
| Music analysis (key, tempo, chords), animal species | Specialised models |

## Don't use

The risk here is in the use itself, not the evidence:

- **Identity verification:** confirming who a speaker is.
- **Detecting minors, or treating demographics as fact:** age, sex, nationality, ethnicity. Accent, perceived age and perceived sex are raters' impressions. Use them to route support (for example offering another language), never to screen, rank or treat people differently.
- **Hidden feelings or sincerity:** whether someone is lying, or how they "really" feel. The API estimates what raters would hear, not what's inside someone.

## Audio it hasn't been checked on

- **Phone audio at 8 kHz.** Accepted, but check thresholds on your own phone recordings.
- **Results on acted or synthetic clips** may not carry over to real calls. Test on real recordings from your domain.
