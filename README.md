# Audio Decisions Skills

Agent skills for the [Audio Decisions API](https://dev.hume.ai/audio-decisions/docs/overview). The API answers questions about audio that you write in plain language. Send a recording with your questions, and each answer comes back as probabilities that estimate how human raters would respond, which your code can threshold, rank, or combine.

The `audio-decisions-api` skill teaches coding agents such as Claude Code, Codex, and Cursor to build with the API: whether a question suits it, which audio each request should cover, how to word questions, and how to read and check the answers. It applies to recordings processed in bulk, to live audio, and to anything in between.

1. [Documentation](https://dev.hume.ai/audio-decisions/docs/overview)
2. [Writing questions](https://dev.hume.ai/audio-decisions/docs/writing-questions)
3. [Primitives](https://dev.hume.ai/audio-decisions/docs/primitives)
4. [Interpreting answers](https://dev.hume.ai/audio-decisions/docs/interpreting-answers)
5. [Cookbooks](https://dev.hume.ai/audio-decisions/cookbooks/overview)
6. [API reference](https://dev.hume.ai/audio-decisions/reference)
7. [Support](https://dev.hume.ai/audio-decisions/docs/support)
8. [Python SDK](https://github.com/HumeAI/hume-audio-decisions-python-sdk)
9. [TypeScript SDK](https://github.com/HumeAI/hume-audio-decisions-typescript-sdk)

## Requirements

1. A coding agent that supports [Agent Skills](https://agentskills.io).
2. A Hume API key, set as the `HUME_API_KEY` environment variable. See [Authentication](https://dev.hume.ai/audio-decisions/docs/authentication).

The skill's examples use the Python SDK, which requires Python 3.10 or later, and the TypeScript SDK, which requires Node.js 18 or later. The Python examples also use [ffmpeg](https://ffmpeg.org) to cut and extract audio.

## Installation

### Skills CLI

The [skills CLI](https://github.com/vercel-labs/skills) installs the skill for Claude Code, Codex, Cursor, and other agents. It requires Node.js 22.20 or later.

```bash
npx skills add https://github.com/HumeAI/hume-audio-decisions-skills/tree/main/audio-decisions-api
```

The CLI asks which agents to install the skill for, and installs it in the current project unless you pass `-g`.

### Claude Code

Copy the `audio-decisions-api` folder into `~/.claude/skills/` to use the skill in every project, or into a project's `.claude/skills/` to share it through that repository. Claude Code picks up the skill in the current session without a restart.

```bash
git clone https://github.com/HumeAI/hume-audio-decisions-skills.git
cp -r hume-audio-decisions-skills/audio-decisions-api ~/.claude/skills/
```

### Other agents

The skill follows the [Agent Skills specification](https://agentskills.io/specification). Copy the `audio-decisions-api` folder to the directory your agent loads skills from.

## Usage

Your agent loads the skill when a task involves the Audio Decisions API. In Claude Code, you can also run it directly with `/audio-decisions-api`. Some prompts to try:

1. "Go through the support calls in `./calls`, flag the ones where the caller sounds frustrated, and write the results to a CSV."
2. "Add a live indicator to our web app that shows whether the speaker sounds confident while they practice a pitch."
3. "Can the Audio Decisions API tell whether a caller is likely to cancel? If not directly, what should we ask it instead?"

## Contents

The skill starts at `SKILL.md`, which points the agent to the other files as it needs them.

| File | Covers |
|---|---|
| [SKILL.md](audio-decisions-api/SKILL.md) | Whether a question suits the API, choosing the audio for each request, writing questions, and reading and checking answers. |
| [references/audio-units.md](audio-decisions-api/references/audio-units.md) | Speakers and channels, window sizing, finding when events occur, long recordings, live audio, and preparing audio. |
| [references/question-design.md](audio-decisions-api/references/question-design.md) | Question types, wording, options, scales, `context`, and example questions by area. |
| [references/capabilities.md](audio-decisions-api/references/capabilities.md) | What generally works, what works under conditions, what is unproven, and what not to use the API for. |
| [references/breaking-down-questions.md](audio-decisions-api/references/breaking-down-questions.md) | Turning an outcome question into listening questions plus other signals. |
| [references/api-reference.md](audio-decisions-api/references/api-reference.md) | The endpoints, requests and responses, counting input tokens, the SDKs and their audio helpers, runs, errors, limits, and workarounds for what the first release leaves out. |
| [examples/bulk-review.md](audio-decisions-api/examples/bulk-review.md) | Many recordings, some over 5 minutes, with answers combined per file and exported (Python). |
| [examples/realtime-mic.md](audio-decisions-api/examples/realtime-mic.md) | Live microphone input with rolling windows in the browser (TypeScript). |
| [examples/call-escalation.md](audio-decisions-api/examples/call-escalation.md) | A stereo support call with per-turn questions and a transcript, built up into "is this call getting heated?" (Python). |

## Updating

The skill was written for the first release of the API. Where it and the [API reference](https://dev.hume.ai/audio-decisions/reference) disagree, the reference is correct. To update the skill, run `npx skills update` for a CLI install, or copy the folder again.

## Contributing

To report a problem with the skill, open an issue in this repository. For help with the API itself, see [Support](https://dev.hume.ai/audio-decisions/docs/support).

## License

[MIT](LICENSE)
