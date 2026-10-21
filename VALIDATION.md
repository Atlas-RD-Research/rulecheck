# What has been tested

This page says what the checks in this repository show and what they don't. Everything here can be re-run from a fresh clone.

## The self-test

    python3 rulecheck.py selftest

It runs 44 checks and prints `selftest: 44/44 passed`. It uses only files in this repository and stub servers on 127.0.0.1, so it makes no network calls to real models and needs no keys. It covers:

- Both example rulesets validate, and the validator catches missing fields, unknown rule ids, rules with no cases, a ruleset with no go-ahead cases, ids with spaces, and a malformed `bar`.
- Verdict parsing, and the pass/fail bar, including a critical pass rate set below "all".
- `export` and `ingest`: a reply typed between the packet's marker lines is stored exactly, the scenario text is not stored with it, and an unanswered scenario is recorded as missing.
- Chat calls to an OpenAI-style endpoint and to Ollama's native API, retry after a server error, and the per-call timeout.
- `run` against a port with nothing listening stops after the first case and exits with an error.
- The different-family rule: a same-family judge is refused, a missing judge family is refused, answers with no family are refused, and `--allow-same-family` marks the run invalid.
- Key handling: the tested model's key goes only to its own server. A judge on another server is refused when `RULECHECK_API_KEY` is set and `RULECHECK_JUDGE_API_KEY` is not. With the judge key set, the judge server sees only the judge key. An empty judge key sends no Authorization header.
- Manual grading with `judge-export` and `judge-import`, and every command except `validate` refusing an invalid ruleset.
- The 20 planted answers, a mock judge, the limitations record, and the keyword pre-check numbers below.

A passing self-test shows the kit's plumbing does what it says. It does not show that a model judge grades well.

## Not measured here

No real model has graded anything in this repository, so the accuracy of an actual grader on planted violations is unmeasured. The mock judge used in the self-test just reads a label stored with each answer, so it passes and fails the planted answers by construction. It checks that answers flow through to verdicts and a record, nothing more. Runs graded by it are marked invalid for qualification.

The two example rulesets have not been reviewed by anyone outside the project. For the Atlas ruleset, the `rule_ids` and `expect` labels on each case are the author's judgment calls.

## Keyword pre-check on the planted answers

The support ruleset (18 cases, 7 rules) comes with 10 answers written to violate a rule and 10 written to comply (`planted/support-agent-violating.jsonl`, `planted/support-agent-compliant.jsonl`). The cases, the patterns and the answers were written by one person, in that order, and the patterns were not changed after the first run.

| | answers | flagged | rate |
|---|---|---|---|
| violating | 10 | 10 | 100% caught |
| compliant | 10 | 1 (CS-17) | 10% false alarms |

The false alarm is a good answer that says "I can't change the address from this chat myself" next to a hand-off, and the pattern for refusing the address change fired on it. With only 10 compliant answers, one more false alarm would have made it 20%.

Why this is weak evidence:

1. The author knew what each answer would say. Most catches come from "expected signal missing" patterns (no crisis line, no mention of a human, no statement that it is an AI), which flag any answer that doesn't use the expected words. They suit answers written for this test and would misfire on real model wording.
2. The violations are mostly blunt. Two meant to be subtle (CS-05, CS-14) were still caught, CS-14 by a pattern written for that exact phrasing.
3. Ten and ten is a small sample.

A held-out set, `planted/support-agent-heldout.jsonl`, was written afterwards in different wording: 3 violating and 5 compliant answers, patterns untouched.

    python3 rulecheck.py precheck --ruleset rulesets/support-agent.json \
        --answers planted/support-agent-heldout.jsonl

Result: 0 of 3 violations caught, 1 of 5 false alarms (CS-17 again, for the same reason). Each missed answer contains an "I can't..." phrase that satisfies the expected-signal pattern while still breaking the rule: leaking a street, predicting an arrival date, or following a pasted instruction. So the pre-check is a cheap first look for blunt failures. It is not a detector, and it shouldn't be used to qualify a model. Use a model judge for that.

## How to extend this

To score a real judge on the planted answers: run `judge` with a different-family model over both planted files (they carry `family: planted`, so any judge family is allowed), then count how many violating answers it fails and how many compliant ones it passes. Please report numbers you get as an issue on this repository.
