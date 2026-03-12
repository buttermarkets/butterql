# butterql

Multi-leg trade planner for bin markets.

butterql is a **skill** — a structured prompt that teaches an
LLM to convert a price thesis into a defined-risk shape
(corridor, wedge, fence, or ladder) on bin markets, then
walk through discovery, planning, confirmation, and execution.

Currently targets **Polymarket** as backend.

## what it does

> "I think BTC stays between 68k and 74k tomorrow"

→ corridor shape → discover bins → scenario table with all
bins → confirm gate → execute

## shapes

| shape | use for |
|---|---|
| corridor | price stays inside a range |
| wedge | directional with close-miss recovery |
| fence | breakout either direction |
| ladder | broad weighted exposure |

## usage

Point your coding agent at `SKILL.md`. The skill works with
any agent that can execute shell commands and read files.

Current backend requires the
[polymarket CLI](https://github.com/Polymarket/polymarket-cli).
Falls back to fixture files when CLI is unavailable.

## structure

```
SKILL.md          # the skill (agent reads this)
fixture/          # offline market data snapshots
test/             # pressure test scenarios (TDD for skills)
```

## testing

`test/butterql.md` contains ~60 test scenarios across 11
dimensions including adversarial multi-pressure cases. These
follow the [Obra/superpowers](https://github.com/obra/superpowers)
TDD-for-skills methodology.

## license

MIT
