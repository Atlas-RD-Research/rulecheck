# rulecheck

rulecheck finds out which of your rules an AI agent breaks. You write the rules and a set of test scenarios. It puts each scenario to the model, has a model from a different family grade the answers, and writes a limitations record: which rules this model broke, how serious each was, and which scenarios showed it.

Two things set it apart from general eval tools. It refuses to run if the grader belongs to the same model family as the model being tested, and it never sends the tested model's API key to the grader's server. The record is a JSON file, so another tool can read it, and there is a short markdown version for people.

It is one Python 3 file with no dependencies and no accounts, tested on Python 3.15. MIT licence. Questions and bugs: open an issue on this repository.

## Who it's for

Anyone who runs an AI agent under rules that matter: a support bot with a refund policy, an agent that can send email, an internal assistant that must not leak data. Use it when you change the model, rewrite the prompt, or widen what the agent is allowed to do, and you want to see what got worse before a customer does.

It tests what a model says it would do, in text. It does not test tool calls or actions. A pass is necessary, not sufficient.

## Quick start

    git clone https://github.com/Atlas-RD-Research/rulecheck
    cd rulecheck
    python3 rulecheck.py selftest
    python3 rulecheck.py validate rulesets/support-agent.json
    python3 rulecheck.py precheck --ruleset rulesets/support-agent.json \
        --answers planted/support-agent-violating.jsonl planted/support-agent-compliant.jsonl

`selftest` runs 44 offline checks against local stub servers and the files in this repository; it should end with `selftest: 44/44 passed`. `validate` checks the example ruleset (18 cases). `precheck` runs the keyword screen over 20 sample answers that ship in `planted/`, no model needed.

To test a real model you need two OpenAI-compatible chat endpoints (or Ollama), one for the model under test and one for the grader, from different model families.

    # 1. get answers from the model under test
    export RULECHECK_API_KEY=...            # only if that endpoint needs a key
    python3 rulecheck.py run --ruleset rulesets/support-agent.json \
        --endpoint http://localhost:8000/v1 --model my-model --family qwen --out answers.jsonl
    # for Ollama's native API instead:
    #   python3 rulecheck.py run --api ollama --num-ctx 8192 --endpoint http://localhost:11434 \
    #       --ruleset rulesets/support-agent.json --model llama3 --family meta --out answers.jsonl

    # 2. grade with a model from a different family
    export RULECHECK_JUDGE_API_KEY=...      # the grader's own key; see "API keys" below
    python3 rulecheck.py judge --ruleset rulesets/support-agent.json --answers answers.jsonl \
        --judge-endpoint http://localhost:9000/v1 --judge-model other-model --judge-family openai \
        --out verdicts.json

    # 3. write the record
    python3 rulecheck.py record --ruleset rulesets/support-agent.json --verdicts verdicts.json \
        --out-json limitations.json --out-md limitations.md

Each case can set its own `timeout`; `--timeout` and `--retries` set the defaults. Failed or empty answers are retried. If they still fail, the case is recorded with the error and counts as a fail. If the server refuses the connection, `run` stops at the first case, and it exits with an error if every case failed.

### API keys

Two environment variables, never files:

- `RULECHECK_API_KEY` is sent only to the server under test (`run`).
- `RULECHECK_JUDGE_API_KEY` is sent only to the grader's server (`judge`).

rulecheck will not send the tested model's key to a different server. If `RULECHECK_API_KEY` is set and the grader is on a different server than the one that answered, `judge` stops until you set `RULECHECK_JUDGE_API_KEY`. Set it to an empty string (`RULECHECK_JUDGE_API_KEY=`) if the grader needs no key. If the grader is the same server that answered, the first key is reused. Answers from `ingest` don't record a server, so they always count as a different one.

### Without a live model

- **A person or another tool answers.** `export` writes a packet of scenarios. For each scenario the packet has two marker lines, `<!-- answer CS-01 -->` and `<!-- end CS-01 -->`. Type the reply between them and leave the markers in. `ingest` stores only what is between the markers, so the scenario text never ends up in the answer. You declare the model name and family.
- **A person or another tool grades.** `judge-export` writes one item per answer with the full grading prompt. Fill in `verdict` (pass or fail) and `reason` on each line, then run `judge-import --judge NAME --judge-family FAMILY`. The family rule below applies to people and tools too: whoever grades must not be the model that answered.
- **A keyword screen with no model at all.** `precheck` flags answers that look like violations using patterns stored in the ruleset. It is quick and cheap and it is not a judge; it cannot qualify a model. VALIDATION.md has the numbers.

## How to write a ruleset

A ruleset is one JSON file. Start from `rulesets/support-agent.json`.

- `name` and `version`, plus an optional `description`. Rule and case ids may use letters, digits, dot, dash and underscore.
- `system_prompt`: the rules as the agent will see them. State the rules; do not hint at answers.
- `rules`: a list of `{id, text, severity}`. Severity is `critical` or `major`.
- `cases`: a list of scenarios. Each has `id`, `rule_ids` (which rules it tests), `scenario` (what the user says, including any lookup data the agent would have), `must` (what a passing answer shows), `must_not` (what fails it), `grading` (one or two sentences for the judge), `severity`, and `expect`.
- `expect` is one of `refuse` (the agent should decline or hold), `comply` (the agent should do the task and follow the rule while doing it), or `go-ahead` (the agent should simply do the work; an agent that refuses is wrong).
- `bar` (optional): `critical` is `"all"` (the default; every critical case must pass) or a minimum pass rate from 0 to 1, and `major_min_rate` is the minimum pass rate for major cases (default 0.9).
- `precheck` (optional, per case): `violation_any` is a list of regexes that mark an answer as a likely violation, and `require_all` is a list of regexes that must each appear in a good answer.

Write the go-ahead cases first. A test where refusing everything passes is useless, and the validator insists on at least one.

`python3 rulecheck.py validate yourfile.json` reports missing fields, bad severities, unknown rule ids, rules that no case tests, duplicate or badly formed ids, a bad `bar`, bad regexes, and a ruleset with no go-ahead cases. Every command except `validate` refuses to run on an invalid ruleset.

## The judge must come from a different family

The model that answers must not be graded by a model of the same family. Models from one family share training data, habits and blind spots, and tend to go easy on answers that read like their own. A same-family judge makes the numbers look better than they are, and you cannot tell by how much.

rulecheck makes you say which family each model belongs to (`--family` when answering, `--judge-family` when judging; use any consistent labels such as `openai`, `qwen`, `anthropic`, `meta`). It refuses to run if they match, or if the answers carry no family at all. `--allow-same-family` lets you run anyway; the output is then marked invalid for qualification and the limitations record says so. The mock judge used in tests is also marked invalid.

The family label is what you declare. rulecheck cannot tell that a fine-tune of one model is the same family as another. Be honest when you declare it. For critical failures, have a person read the judge's reasoning, since judges misread nuance.

## What you get

`limitations.json` has: the model and family, the ruleset name, version and a content hash, the judge and its family, the date, the pass/fail result against the bar, and `broken_rules`: for each broken rule, its text, its severity, the scenario ids that failed, the severity of each, and the judge's reason. Cases with no usable answer are listed separately and count as failures. If judging has not happened, the record says `judging_pending` and has no result.

## Examples

- `rulesets/support-agent.json`: a generic customer-support ruleset (18 cases, 4 go-ahead, 7 rules). The company and customers are invented.
- `rulesets/atlas-decision-regression.json`: 29 cases (21 refuse, 4 comply, 4 go-ahead; 12 rules) that Atlas Research uses to check its own AI agents against its operating rules. Each case has a `source_rule` field with its original label. It shows a bigger ruleset with long, realistic scenarios.
- `planted/`: canned answers for the support ruleset, labelled violating or compliant, used to test the pre-check and the pipeline. `support-agent-heldout.jsonl` was written afterwards, in different wording.
- `examples/build_support.py` regenerates the support ruleset and all three planted files. Run it from anywhere; it writes into this repository.

## Limits

- Text only, not actions. A model that says the right thing may still act wrongly with tools.
- 18 or 29 cases is a screen, not proof. Add a case whenever a real incident shows a gap.
- Judges make mistakes, including across families. Keep the grading text tight and read the critical failures.
- Anything you publish becomes training data. Rotate cases.
- The pre-check is a keyword screen written by the same person who wrote the test answers; see VALIDATION.md.
