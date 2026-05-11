---
title: 'Benchmark Sapat Transcription Providers'
description:
  'Compare Sapat transcription providers in Daytona and choose the right route for each recording type.'
date: 2026-05-11
author: 'Dustin Vali'
tags: ['ai', 'transcription', 'daytona']
---

# Benchmark Sapat Transcription Providers

Most transcription guides stop after the first successful transcript. That is useful for a demo, but it is not enough for an engineering workflow.

If your team uses transcripts for customer calls, design reviews, QA videos, sales demos, or incident reports, the provider choice affects turnaround time, transcript quality, operating cost, and the amount of cleanup work someone has to do later.

Sapat is a small Python command-line tool that gives you a clean way to compare providers. It converts `.mp4` files to MP3 with `ffmpeg`, sends the audio to a selected transcription provider, and writes a `.txt` transcript next to the source video.

Its current source supports OpenAI, Groq Cloud, and Azure OpenAI through a required `--api` option.

This guide shows how to run a provider benchmark inside a [Daytona workspace](/definitions/20240819_definition_daytona%20workspace.md).

You will create a repeatable test folder, run the same clips through each provider, capture the practical differences, and turn the result into a provider routing policy your team can use.

![Sapat provider benchmark workflow](/assets/20260511_benchmark_sapat_transcription_providers_img1.png)

## TL;DR

- Use Sapat to run the same representative `.mp4` files through `--api openai`, `--api groq`, and `--api azure`.
- Keep the first benchmark simple: one workspace, one fixture folder, one scorecard, and the same `--quality`, `--language`, `--prompt`, and `--temperature` values for every run.
- Score providers on more than transcript text. Track setup friction, latency, file-size limits, correction behavior, language handling, and post-editing effort.
- Convert benchmark results into a routing policy: fast drafts, high-accuracy review, regulated workloads, multilingual media, or low-cost batch work.

## What Sapat Actually Does

Before benchmarking, it helps to know where Sapat makes decisions.

The CLI takes a file or a directory path. If you pass one file, Sapat processes that file. If you pass a directory, it loops over the `.mp4` files in that directory.

For each video, it creates a temporary MP3 with `ffmpeg`, sends that MP3 to the provider selected by `--api`, writes a `.txt` file with the transcript, and removes the temporary MP3.

The current command shape is:

```bash
sapat <video_file_or_directory> \
  --api openai \
  --quality M \
  --language en \
  --prompt "Product names: Daytona, Sapat" \
  --temperature 0.3
```

The provider can be `openai`, `groq`, or `azure`. The `--quality` flag controls the MP3 conversion settings:

| Quality | Audio settings used by Sapat | Useful starting point |
| --- | --- | --- |
| `L` | 22050 Hz, mono, 96 kbps | Long internal recordings where speed and file size matter |
| `M` | 44100 Hz, mono, 96 kbps | Default benchmark setting |
| `H` | 44100 Hz, stereo, 192 kbps | Short clips, noisy clips, or quality-sensitive review |

Sapat also accepts `--language`, `--prompt`, and `--temperature`. Keep these values stable across providers during the benchmark. If you change the prompt or temperature while switching providers, you will not know whether the difference came from the provider or the test setup.

There is also a `--correct` flag that asks a chat model to clean up the transcript. Treat that as a second pass. First benchmark raw transcription. Then benchmark correction separately, because correction can hide transcription errors and add another model, another deployment, and another cost path.

## Create a Daytona Workspace

A benchmark is only useful if someone else can repeat it. Daytona helps because the workspace captures the same repository, tools, shell history, and environment shape for every run.

Start from the Sapat repository:

```bash
daytona create https://github.com/nkkko/sapat --code
```

Inside the workspace, install the project dependencies and confirm that `ffmpeg` is available:

```bash
python --version
python -m pip install -e .
ffmpeg -version
sapat --help
```

If `ffmpeg` is missing, install it with the package manager available in your workspace image. The exact command depends on the base image, but Debian and Ubuntu based images commonly use:

```bash
sudo apt-get update
sudo apt-get install -y ffmpeg
```

Keep the benchmark workspace separate from production transcripts. That prevents credentials, fixture files, and test notes from being mixed into a regular project.

## Prepare a Fair Test Set

Do not benchmark with one perfect clip. Real transcription systems fail in boring ways: a speaker moves away from the microphone, a product name sounds like a common word, a customer says a number too quickly, or two people talk over each other.

Create a small fixture folder:

```bash
mkdir -p benchmark/clips benchmark/results
```

Use three to five short `.mp4` files. A good first set looks like this:

| Fixture | Duration | Why it matters |
| --- | --- | --- |
| `clean_demo.mp4` | 30-90 seconds | Baseline accuracy for a clear speaker |
| `meeting_overlap.mp4` | 60-120 seconds | Tests interruptions and crosstalk |
| `domain_terms.mp4` | 30-90 seconds | Tests product names, acronyms, and API terms |
| `noisy_clip.mp4` | 30-90 seconds | Tests background noise and compression |
| `non_english_or_accent.mp4` | 30-120 seconds | Tests language hints and accent handling |

Avoid private customer data for the first benchmark. If you must use sensitive media later, run the benchmark under your team's normal secrets and data-handling rules. Provider benchmarking is still data processing.

## Configure Provider Credentials

Sapat reads provider settings from a `.env` file. Put only the providers you plan to test in the first pass.

```bash
cp .env.example .env 2>/dev/null || touch .env
```

For OpenAI, Sapat expects:

```bash
OPENAI_API_KEY=your_openai_api_key
OPENAI_MODEL=whisper-1
OPENAI_API_ENDPOINT=https://api.openai.com/v1/audio/transcriptions
OPENAI_MODEL_NAME_CHAT=gpt-4o
```

For Groq Cloud, Sapat expects:

```bash
GROQCLOUD_API_KEY=your_groq_api_key
GROQCLOUD_MODEL=whisper-large-v3-turbo
GROQCLOUD_API_ENDPOINT=https://api.groq.com/openai/v1/audio/transcriptions
GROQCLOUD_MODEL_NAME_CHAT=llama3-8b-8192
```

For Azure OpenAI, Sapat expects:

```bash
AZURE_OPENAI_API_KEY=your_azure_api_key
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com
AZURE_OPENAI_DEPLOYMENT_NAME_WHISPER=whisper
AZURE_OPENAI_API_VERSION_WHISPER=2024-06-01
AZURE_OPENAI_DEPLOYMENT_NAME_CHAT=gpt-4o
AZURE_OPENAI_API_VERSION_CHAT=2023-03-15-preview
```

Two setup details matter during a benchmark.

First, keep provider credentials out of Git. A Daytona workspace can make setup repeatable without committing secrets.

Second, write down which model or deployment each provider uses. Comparing a fast Groq Whisper model against a different Azure deployment may still be useful, but it is not a pure provider comparison unless the model class and deployment intent are clear.

## Run the Same Inputs Through Each Provider

Use the same flags for every provider. Start with medium MP3 quality, English language hints, a domain prompt, and low temperature.

```bash
export SAPAT_PROMPT="Product names: Daytona, Sapat, workspace, provider benchmark"

for api in openai groq azure; do
  mkdir -p "benchmark/results/$api"
  /usr/bin/time -p sapat benchmark/clips \
    --api "$api" \
    --quality M \
    --language en \
    --prompt "$SAPAT_PROMPT" \
    --temperature 0.3

  mv benchmark/clips/*.txt "benchmark/results/$api"/
done
```

If you want more precise timing, wrap each run in a small shell script that records start and end timestamps per file. The simple `time` command is enough for a first pass, but per-file timing is better when the fixture set contains clips of different lengths.

Here is a minimal timing wrapper you can adapt:

```bash
#!/usr/bin/env bash
set -euo pipefail

api="$1"
clip="$2"
name="$(basename "$clip" .mp4)"
mkdir -p "benchmark/results/$api"

started_at="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
start_seconds="$(date +%s)"

sapat "$clip" \
  --api "$api" \
  --quality M \
  --language en \
  --prompt "$SAPAT_PROMPT" \
  --temperature 0.3

end_seconds="$(date +%s)"
ended_at="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
duration_seconds="$((end_seconds - start_seconds))"

mv "${clip%.mp4}.txt" "benchmark/results/$api/$name.txt"
printf "%s,%s,%s,%s,%s\n" "$api" "$name" "$started_at" "$ended_at" "$duration_seconds" >> benchmark/results/timing.csv
```

Run it like this:

```bash
chmod +x benchmark/run_one.sh

for api in openai groq azure; do
  for clip in benchmark/clips/*.mp4; do
    benchmark/run_one.sh "$api" "$clip"
  done
done
```

This does not need to be fancy. The goal is to leave enough evidence that a teammate can understand how you chose the default provider.

## Score the Transcripts Like an Engineer

Automated word error rate is useful when you have a clean reference transcript. Most teams do not. For a practical first benchmark, use a manual scorecard that focuses on the transcript defects that slow engineering work.

Create `benchmark/results/scorecard.md`:

```markdown
| Clip | Provider | Latency | Names and acronyms | Numbers | Speaker turns | Cleanup needed | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| clean_demo | openai |  |  |  |  |  |  |
| clean_demo | groq |  |  |  |  |  |  |
| clean_demo | azure |  |  |  |  |  |  |
```

Use a simple scale for the qualitative columns:

- `3`: Ready to use with light editing.
- `2`: Useful, but needs a review pass.
- `1`: Important details are missing or wrong.
- `0`: Failed, empty, or unusable output.

Read the transcripts next to the source clip. Do not only check grammar. Look for engineering details:

- Product names and acronyms: Did the provider keep project-specific vocabulary intact after the `--prompt` hint?
- Numbers and commands: Did it preserve ports, versions, timestamps, issue IDs, and command names?
- Speaker transitions: Did overlapping comments become confusing?
- Hallucinated cleanup: Did optional correction make the text nicer while changing meaning?
- Operational friction: Did the provider require extra deployment setup, hit file-size limits, or fail on a clip others handled?

You can add a checksum or file size column if you want to track transcript changes over time. That helps when you rerun the benchmark after changing audio quality, prompts, or provider models.

## Compare Raw Transcription and Correction Separately

Sapat's `--correct` flag can be valuable for turning rough transcripts into readable notes. It can also make a benchmark misleading.

Run raw transcription first:

```bash
sapat benchmark/clips/clean_demo.mp4 \
  --api openai \
  --quality M \
  --language en \
  --prompt "$SAPAT_PROMPT" \
  --temperature 0.3
```

Then run a second correction pass and store it under a different folder:

```bash
mkdir -p benchmark/results/openai_corrected

sapat benchmark/clips/clean_demo.mp4 \
  --api openai \
  --quality M \
  --language en \
  --prompt "$SAPAT_PROMPT" \
  --temperature 0.3 \
  --correct

mv benchmark/clips/clean_demo.txt benchmark/results/openai_corrected/clean_demo.txt
```

Score corrected transcripts with a different question: "Did this reduce editor time without changing meaning?" A transcript can look polished and still be worse for an engineer if it changes a version number, normalizes a command incorrectly, or drops an uncertain phrase.

## Turn Results Into a Routing Policy

The output of this benchmark should not be a vague preference. Write down a routing policy that someone can apply before starting a transcription job.

Use a short table:

| Workload | Default provider | Flags | Review rule |
| --- | --- | --- | --- |
| Internal demos under 10 minutes | Groq | `--quality M --temperature 0.3` | Spot-check names and commands |
| Customer calls with product terms | OpenAI | `--quality H --prompt "$SAPAT_PROMPT"` | Full human review before sharing |
| Regulated workspace data | Azure | Team-approved deployment | Follow internal data policy |
| Noisy field recordings | Best benchmark score | `--quality H` | Compare first minute before batch run |
| Low-cost backlog transcription | Fastest acceptable provider | `--quality L` or `M` | Sample every fifth transcript |

The provider names in this table are placeholders. Replace them with your actual benchmark outcome. The important part is the decision shape: choose by workload, not by habit.

Keep the policy close to the team that uses it. A small `docs/transcription-routing.md` file in the project repository is often enough. Include the benchmark date, fixture descriptions, model or deployment names, and the person who ran the test.

## Troubleshooting

**Problem: Sapat says the API option is missing.**

Use `--api openai`, `--api groq`, or `--api azure`. The provider is required.

**Problem: The transcript file is empty or missing.**

Check provider credentials and endpoint values in `.env`. Then run one short file instead of the whole folder so the failure is easier to inspect.

**Problem: `ffmpeg` fails before the provider call.**

Confirm `ffmpeg -version` works in the Daytona workspace. If the video file is unusual, convert it manually once and inspect the output.

**Problem: One provider fails on larger files.**

Shorten the fixture, lower `--quality`, or split the source video before sending it. Sapat's OpenAI and Groq implementations validate audio size before upload.

**Problem: The prompt seems ignored.**

Use a shorter prompt with only domain words that matter. Prompt hints are not a glossary guarantee, so score the result instead of assuming the hint worked.

## Conclusion

A transcription provider benchmark does not need a large evaluation platform. You need representative clips, stable flags, a repeatable Daytona workspace, and a scorecard that reflects the work your team actually does after transcription.

Sapat is a good fit for that loop because it keeps provider selection explicit. Once you know which provider works best for each recording type, you can stop arguing about the default and start routing jobs with evidence.

## References

- [Sapat repository](https://github.com/nkkko/sapat)
- [Daytona project](https://github.com/daytonaio/daytona)
- [FFmpeg documentation](https://ffmpeg.org/documentation.html)
- [OpenAI audio transcription API reference](https://platform.openai.com/docs/api-reference/audio)
- [Groq speech-to-text documentation](https://console.groq.com/docs/speech-to-text)
- [Azure OpenAI documentation](https://learn.microsoft.com/azure/ai-services/openai/)
