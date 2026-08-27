# Unfog

**Turn vague asks into evidence-backed execution contracts.**

Unfog is an Agent Skill for people who communicate in compressed directions—“build it,” “fix this,” “make it useful”—and expect an agent to recover the real target, preserve what matters, execute the full chain, and prove the result.

Read why it exists: [I Built Unfog Because “Write a Better Spec” Is Usually the Wrong Advice](https://www.linkedin.com/pulse/i-built-unfog-because-write-better-spec-usually-wrong-justin-fowler-awysc/)

It does not inflate every request into a formal specification. It silently compiles non-trivial, underspecified work into a small execution contract:

- the outcome someone will actually observe;
- the authoritative target and running artifact;
- the factual premise behind the request;
- the affected population and preservation boundaries;
- observable acceptance probes and controls; and
- the delivery chain required to finish.

Unfog asks a question only when a genuinely material fork remains. Otherwise, it uses evidence, chooses the least-expanding reversible interpretation, and keeps moving.

## Install

```sh
git clone https://github.com/justinfowler925/unfog.git
python3 unfog/skill/scripts/install.py --replace --remove-legacy
```

The installer links one canonical skill directory into the user-level skill locations for:

| Agent | Install path |
|---|---|
| Codex | `~/.codex/skills/unfog` |
| Cursor | `~/.cursor/skills/unfog` |
| Claude | `~/.claude/skills/unfog` |
| Shared Agent Skills | `~/.agents/skills/unfog` |

Restart any already-open agent after installation so it refreshes its skill catalog.

## Invoke it

Automatic invocation is enabled. Unfog should load for non-trivial build, change, diagnose, or review requests whose target, scope, acceptance criteria, environment, or delivery details are missing.

Force it explicitly with:

```text
$unfog build this
$unfog fix what this conversation exposed
$unfog turn this direction into a shipped, verified result
```

It should not activate for narrow factual questions or obvious one-line edits.

## Optional machine-checked contracts

Most work can keep the contract in agent working notes. For long-running, handed-off, or risky work, the bundled helper creates and validates a JSON contract:

```sh
python3 skill/scripts/intent_contract.py new \
  --request "build it" \
  --mode change \
  --output /tmp/unfog-contract.json

python3 skill/scripts/intent_contract.py validate \
  /tmp/unfog-contract.json \
  --phase ready
```

The validator refuses a zero denominator, unresolved material fork, missing user-observable probe, missing control, or incomplete delivery receipt.

## Why this exists

The cost of a vague request is rarely the missing prose. It is the cycles spent asking questions that evidence could answer, implementing against the wrong artifact, silently widening scope, or calling work complete without a user-observable receipt.

Unfog treats ambiguity as something to compile—not something to complain about.

## License

MIT
