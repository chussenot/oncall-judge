# oncall-judge

Typed, calibrated verdicts on production log excerpts from a small local
model: which of twelve incidents a log describes and how severe it is,
served by `transformers serve` and read through a [`jud`](https://github.com/chussenot/judgment)
rubric whose gates say when to page.

The model is [`rachitst/oncall-ai-incident-classifier`](https://huggingface.co/rachitst/oncall-ai-incident-classifier),
a Qwen2.5-0.5B-Instruct fine-tune (QLoRA, merged) that answers a fixed system
prompt with `{"incident_type": "...", "severity": "P1|P2|P3|P4"}`. It runs on
a CPU in a few seconds per request.

## Setup

[mise](https://mise.jdx.dev) installs the two tools and runs the tasks:

```sh
mise trust && mise install      # transformers[serving,torch] and jud
```

`jud` reads its backend from `~/.config/jud/config.yaml`; there is no
environment variable for the model name, so the file is required:

```yaml
base_url: http://localhost:8000              # the client appends /v1/systemone
model: rachitst/oncall-ai-incident-classifier
api_key: unused                              # the server ignores it; jud refuses to run without one
timeout_secs: 60
```

`jud config` prints what a run will use.

## Use

```sh
mise run serve                                        # loads the model, listens on :8000
jud incident.jud < payloads/oom-kubernetes.json       # one verdict per question
mise run classify payloads/oom-kubernetes.json        # the same log on the chat endpoint, prints the JSON label
```
<img width="1904" height="504" alt="image" src="https://github.com/user-attachments/assets/53560200-1238-48fd-ac7c-0afde65a78e6" />

A verdict, for the TLS case under `incident-cases.jud`:

```json
{
  "incident_type": {"verdict": "option", "key": "tls_certificate_expired", "confidence": 0.99},
  "severity": {"verdict": "level", "index": 3, "label": "P1", "confidence": 0.99, "reached": true}
}
```

`reached` is the policy's `level_at_least: P2`: true means page the on-call.
Below the confidence bar a verdict is `deferred` for a person to read.

The server speaks two wires. `POST /v1/chat/completions` generates the JSON
label the model was trained to write; `mise run classify` wraps a log in the
training prompt it reads from `decision.json`. `POST /v1/systemone` is what `jud` calls: no text is generated,
each allowed answer is one token and its probability is read from a single
forward pass.

## Files

| Path | What |
|---|---|
| `mise.toml` | tools and the `serve` and `classify` tasks |
| `decision.json` | `--decision-config` for `transformers serve`: a chat template that renders every System One question in the model's training format, and the token scored per answer |
| `incident.jud` | the rubric: a `choice` over the twelve incident types, a `score` over P4 to P1, and the tuned gates |
| `incident-cases.jud` | eight labelled log excerpts, one per severity and eight of the twelve types |
| `recordings/` | the model's recorded answers to those cases, so `jud eval` and `jud tune` run with no server |
| `payloads/` | one `{"log": "..."}` per sample: the logs to judge, with no prompt in them. `jud` reads one on stdin, `mise run classify` wraps one in the chat prompt |

## Why decision.json

Out of the box the System One endpoint asks `A. ... B. ...` and scores the
letters, and this model, trained to emit JSON after one exact system prompt,
answers near chance that way. `decision.json` renders each question as the
training prompt and ends on the assistant prefix `{"incident_type": "` or
`{"severity": "P`, so the next token is the model's own decision. The
`labels` are the first token of each incident type, in the order
`incident.jud` lists them, and the digit after `P` for the levels.

On the eight cases:

| | letter prompt | with `decision.json` |
|---|---|---|
| incident type | 4/8 | 8/8 |
| severity | 1/8 | 6/8, the gate defers the 2 misses |

## Test, record, tune

```sh
jud check incident.jud incident-cases.jud recordings/*.jud   # read every document as the crate does
jud eval --replay recordings incident.jud incident-cases.jud # grade the recordings, no server
jud record --out recordings --refresh incident.jud incident-cases.jud   # ask the server again
jud tune --replay recordings incident.jud incident-cases.jud # propose the gates
```

The bars in `incident.jud` are what `jud tune` read off eight cases, which it
rightly calls a guess with a number on it; add cases and re-tune before
trusting them. `level_at_least: P2` is a policy choice, not a tuned one.

## Caveats

- The `labels` in `decision.json` are positional: reorder the options in
  `incident.jud` and the answers are silently mislabelled.
- The severity prefix `{"severity": "P` never appeared in training; the
  model always wrote the incident type first. It still reads the right level
  on 6 of 8 cases.
- The PyPI `torch` wheel is the CUDA build (5.4 GB) even on a machine
  without an NVIDIA GPU. The CPU wheel index carries no upload dates, which
  mise's release-age cutoff rejects.
- The server warns about unauthenticated Hub requests; set `HF_TOKEN` for
  higher rate limits.
