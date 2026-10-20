# rulecheck

A small kit for finding out which of your rules an AI agent breaks.

You write down the rules the agent must follow and a set of scenarios that test them. rulecheck puts each scenario to the model, has a model from a different family grade the answers, and writes a limitations record: which rules this model broke, how serious each one was, and which scenarios showed it. The record is a JSON file, so another tool can read it. There is a short markdown version for people.

Version 0.1, internal only. Python 3 standard library, nothing to install, no accounts. Status against its own test is in VALIDATION.md.

## Who it's for

Anyone who runs an AI agent under rules that matter: a support bot with a refund policy, an agent that can send email, an internal assistant that must not leak data. Use it when you change the model, rewrite the prompt, or widen what the agent is allowed to do, and you want to see what got worse before a customer does.

It tests what a model says it would do in text. It does not test tool calls or actions. A pass is necessary, not sufficient.

## Quick start

    cd products/rulecheck
    python3 rulecheck.py selftest                       # offline checks of the kit itself
    python3 rulecheck.py validate rulesets/support-agent.json

    # 1. get answers (OpenAI-compatible server, or Ollama's native API)
    python3 rulecheck.py run --ruleset rulesets/support-agent.json \
        --endpoint http://localhost:8000/v1 --model my-model --family qwen --out answers.jsonl
    python3 rulecheck.py run --api ollama --num-ctx 8192 --endpoint http://localhost:11434 \
        --ruleset rulesets/support-agent.json --model llama3 --family meta --out answers.jsonl

    # 2. grade with a model from a different family
    python3 rulecheck.py judge --ruleset rulesets/support-agent.json --answers answers.jsonl \
        --judge-endpoint http://localhost:9000/v1 --judge-model other-model --judge-family openai \
        --out verdicts.json

    # 3. write the record
    python3 rulecheck.py record --ruleset rulesets/support-agent.json --verdicts verdicts.json \
        --out-json limitations.json --out-md limitations.md

An API key, if the endpoint needs one, is read from the environment variable `RULECHECK_API_KEY`. Do not put keys in files. Each case can set its own `timeout`; `--timeout` and `--retries` set the defaults. Failed or empty answers are retried, and if they still fail the case is recorded with the error and counts as a fail.

### Without a live model

- **A person or another tool answers.** `export` writes a packet of scenarios; whoever answers puts each reply under its `## <id>` heading; `ingest` turns that into the answers file. You declare the model name and family.
- **A person or another tool grades.** `judge-export` writes one item per answer with the full grading prompt. Fill in `verdict` (pass or fail) and `reason` on each line, then `judge-import --judge NAME --judge-family FAMILY`. The family rule below applies to people and tools too: whoever grades must not be the model that answered.
- **A keyword screen with no model at all.** `precheck` flags answers that look like violations using patterns stored in the ruleset. It is quick and cheap and it is not a judge; it cannot qualify a model. See VALIDATION.md for how well it works.

## How to write a ruleset

A ruleset is one JSON file. Start from `rulesets/support-agent.json`.

- `name`, `version`, `description`.
- `system_prompt`: the rules as the agent will see them. State the rules; do not hint at answers.
- `rules`: a list of `{id, text, severity}`. Severity is `critical` or `major`.
- `cases`: a list of scenarios. Each has `id`, `rule_ids` (which rules it tests), `scenario` (what the user says, including any lookup data the agent would have), `must` (what a passing answer shows), `must_not` (what fails it), `grading` (one or two sentences for the judge), `severity`, and `expect`.
- `expect` is one of `refuse` (the agent should decline or hold), `comply` (the agent should do the task and follow the rule while doing it), or `go-ahead` (the agent should simply do the work; an agent that refuses is wrong).
- `bar` (optional): the default is every critical case passes and at least 90% of major cases pass.
- `precheck` (optional, per case): `violation_any` is a list of regexes that mark an answer as a likely violation, and `require_all` is a list of regexes that must each appear in a good answer.

Write the go-ahead cases first. A test where refusing everything passes is useless, and the validator insists on at least one.

`python3 rulecheck.py validate yourfile.json` reports missing fields, bad severities, unknown rule ids, rules that no case tests, duplicate ids, bad regexes, and a ruleset with no go-ahead cases. Every other command refuses to run on an invalid ruleset.

## The judge must come from a different family

The model that answers must not be graded by a model of the same family. Models from one family share training data, habits and blind spots, and tend to go easy on answers that read like their own. A same-family judge makes the numbers look better than they are, and you cannot tell by how much.

rulecheck makes you say which family each model belongs to (`--family` when answering, `--judge-family` when judging; use any consistent labels such as `openai`, `qwen`, `anthropic`, `meta`). It refuses to run if they match, or if the answers carry no family at all. `--allow-same-family` lets you run anyway; the output is then marked invalid for qualification and the limitations record says so. The mock judge used in tests is also marked invalid.

The family label is what you declare. rulecheck cannot tell that a fine-tune of one model is the same family as another. Be honest when you declare it. For critical failures, have a person read the judge's reasoning, since judges misread nuance.

## What you get

`limitations.json` has: the model and family, the ruleset name, version and a content hash, the judge and its family, the date, the pass/fail result against the bar, and `broken_rules`: for each broken rule, its text, its severity, the scenario ids that failed, the severity of each, and the judge's reason. Cases with no usable answer are listed separately and count as failures. If judging has not happened, the record says `judging_pending` and has no result.

## Examples

- `rulesets/atlas-decision-regression.json`: Atlas's own 29 cases, converted from `benchmarks/decision-regression/` with scenarios, must and must_not lists, grading text and severities unchanged. `runs/atlas-gpt-oss-20b/` holds the converted gpt-oss:20b answers and a pending record.
- `rulesets/support-agent.json`: a new, generic customer-support ruleset (18 cases, 4 go-ahead). The company and customers are invented.
- `planted/`: canned answers for the support ruleset, labelled violating or compliant, used to test the pre-check and the pipeline. `examples/` has the scripts that regenerate the rulesets and answer files.

## Limits

- Text only, not actions. A model that says the right thing may still act wrongly with tools.
- 18 or 29 cases is a screen, not proof. Add a case whenever a real incident shows a gap.
- Judges make mistakes, including across families. Keep the grading text tight and read the critical failures.
- Anything you publish becomes training data. Rotate cases.
- The pre-check is a keyword screen written by the same person who wrote the test answers; see VALIDATION.md.
