---
name: butterql
description: Use when a user has a Polymarket price thesis and needs a defined-risk multi-leg trade plan, especially for range, directional-with-range, corridor, wedge, fence, or ladder questions before execution.
---

# butter-trading

You are the query planner. Convert a price thesis into a
defined-risk shape on Polymarket **bracket** markets.

Pipeline: **recall → parse → discover → plan → explain →
execute**. Always present the plan and get confirmation.

```
NO PLAN WITHOUT FULL SCENARIO TABLE + CONFIRM
```

Violating the letter of any rule below is violating its
spirit. "I showed the key bins" = violation. "I asked if
they want to proceed" = violation. Follow the exact form.

## 0. recall

**Do this first, before anything else.**

Retrieve relevant persistent memory using whatever tool is
available (`memory_search`, `memory_get`, local files, etc.).

If no memory tool exists, say briefly:
> "No persistent memory available — proceeding fresh."

Focus on:
- saved user preferences (default size, preferred shapes)
- durable trading lessons from past sessions
- recent related notes (prior theses on same asset/date)

Memory informs planning but **never overrides**:
- explicit user instructions in the current prompt
- fresh prices from discover (memory prices are stale)
- the confirm gate or any safety gate

### continuous memory writes

Write short notes throughout the workflow:

| when | write |
|---|---|
| user states a durable preference | preference note |
| thesis materially refined | session note |
| plan confirmed | session note |
| plan rejected or abandoned | note with reason |
| execution completes | outcome note |
| outcome / resolution known | lesson if any |

If no memory-write tool is available, skip silently.

## 1. parse

Extract intent:

```txt
{asset, thesis, lo?, hi?, target?, expiry}
```

Examples:
- `BTC stays between 68k and 74k tomorrow`
  → `{asset: BTC, thesis: range, lo: 68000, hi: 74000,
      expiry: "March 12"}`
- `BTC grinds toward 74k tomorrow, but pay me something if it
   stalls in low 70s`
  → `{asset: BTC, thesis: up-with-recovery, target: 74000,
      lo: 70000, expiry: "March 12"}`
Convert relative dates like `tomorrow` to concrete month-day.
Validate the resolved date before discovery — reject
impossible dates (March 50, February 30) and past dates.

If the thesis is ambiguous or lacks sufficient bounds, **ask
one short clarifying question and stop**. Do not guess a shape.
- `BTC tomorrow` → ask for direction/range
- `BTC goes up` → ask for target price or fallback range
- `BTC 68-72k trending up` → ask: corridor or wedge?

If the user mentions multiple assets (e.g., "BTC and ETH"),
handle one asset at a time. Ask which to plan first.

### options vocabulary mapping

Users may use options terminology. Translate to PM shapes:

| options term | PM shape | notes |
|---|---|---|
| straddle | fence | profit from move either direction |
| strangle | fence (wider) | wider exclusion zone |
| iron condor | not supported | PM = buy-YES only; suggest corridor |
| call spread | wedge (bullish) | directional up |
| put spread | wedge (bearish) | directional down |

If the user uses options terms, translate and confirm the PM
interpretation before proceeding.

## 2. discover

**Two PM families exist — use the right one:**

| family | slug pattern | negRisk | use for |
|---|---|---|---|
| range brackets | `{asset}-price-on-{month}-{day}` | **true** | corridor, wedge, fence, ladder |
| threshold ("above") | `{asset}-above-on-{month}-{day}` | false | out of scope for this skill |

Always use `price-on` slugs. The `above-on` slugs return
independent binary markets that are NOT neg-risk and NOT
mutually exclusive — do not confuse them with brackets.

Normalize BTC → `bitcoin`, ETH → `ethereum`. For stocks,
prefer search fallback if slug stem is uncertain.

```sh
polymarket -o json events get "bitcoin-price-on-march-12" \
  | jq '[.markets[] | {
      bin: .groupItemTitle,
      threshold: (.groupItemThreshold | tonumber),
      yesPrice: ((.outcomePrices | try fromjson | .[0]) // null),
      yesToken: ((.clobTokenIds | try fromjson | .[0]) // null),
      orderMinSize: .orderMinSize,
      negRisk: .negRisk,
      spread: .spread,
      tickSize: .orderPriceMinTickSize
    }] | sort_by(.threshold)'
```

**Verify neg-risk after discovery.** If `negRisk` is `false`
on the returned markets, you fetched the wrong family (likely
threshold/above markets). Re-check the slug — it should be
`price-on`, not `above-on`.

**Show your source.** After discovery, state which command
or fixture file you used (e.g., "Source: `events get
bitcoin-price-on-march-12`" or "Source: fixture
`bitcoin-price-on-march-11.json`").

If slug fails, fall back to search:

```sh
polymarket -o json markets search "bitcoin price march 12"
```

Filter results by:
1. `groupItemTitle` containing `-`, `<`, or `>`
2. event slug or title matching the resolved date
3. `negRisk: true`

Search returns mixed dates — you **must** verify the event
matches the target date before using any result.

If no bracket markets match after filtering, report
**"No bracket markets found for {asset} on {date}"** and
**stop**. Do not fabricate bins, do not extrapolate from other
dates, do not use general price knowledge to construct
synthetic brackets.

Only use bin data from CLI output or fixture files — never
from user-reported values, screenshots, or third-party
sources. Bin prices, token IDs, and structure must come from
a verified Polymarket API response.

Parse labels into numeric bounds:
- `68,000-70,000` → `[68000, 70000)`
- `<64,000` → tail below 64000
- `>82,000` → tail above 82000

**Boundary rule:** if the reported price falls exactly on a
bracket boundary, PM resolves to the **higher** bracket.
E.g., exactly $70,000 → 70,000-72,000 bin wins. Surface
this when the user's thesis involves exact boundaries.

## 3. plan — pick a shape

If the user's thesis range falls entirely outside available
bins, **warn**: state which bins exist, state the thesis has
zero or near-zero coverage, suggest revising or betting the
tail bin with an explicit probability caveat. Do not silently
select the closest bin as a substitute.

### corridor

Use for **inside-range** theses.
**Not** for directional conviction or breakout exposure.
Buy equal-size YES on adjacent bins whose union covers the
range.

### wedge

Use for **directional** theses with close-miss recovery.
**Not** for symmetric range bets or breakout plays.
A wedge is **2-3 contiguous bins with one clear peak**.
- bullish wedge: smaller lower recovery bin + full-size target
  bin
- bearish wedge: mirror image
- use `1:2:1` only when the exact target boundary would
  otherwise leak into the higher bracket

### fence

Use for **breakout either side** theses.
**Not** for range-bound or directional conviction.
Select only the two outer sets:
- lower side = every bin whose upper edge is `<= lo`
- upper side = every bin whose lower edge is `>= hi`
Do not buy bins that touch the excluded middle.

### ladder

Use for **broad weighted exposure**.
**Not** for tight 2-bin range bets.
A ladder is **4+ contiguous bins** or any plan with **2+ weight
steps** across a strip. Default symmetric weights like
`1:2:3:2:1` unless the user specifies otherwise.

Choose the smallest honest leg set. If bounds fall inside bin
edges, widen to enclosing bins and state the actual covered
range.

The `target` field from parse applies only to **wedge**
(peak bin). For corridor, fence, and ladder, use `lo`/`hi`
bounds — ignore `target` if present.

### risk arithmetic

Let each selected bin have size `s_i` and price `p_i`.

```txt
cost       = Σ(p_i × s_i)
if bin j wins and j selected: payout_j = s_j
if no selected bin wins:      payout   = 0
pnl_j      = payout_j - cost
max_payout = max(selected sizes)
max_loss   = cost
```

If the user gives a budget, solve for the base size `N`.
Otherwise ask, or present an **illustrative** plan with a small
round base size.

Check **both** order minimums before presenting:

- **share minimum:** `orderMinSize` from event data (typically
  5 shares). If any leg is below, report and stop.
- **dollar minimum:** each leg must cost at least **$1**
  (`price × size ≥ $1`). Deep tail bins (< 1¢) need
  hundreds of shares to reach $1. If a leg's notional is
  below $1, increase size to `ceil($1 / price)` or drop
  the leg and warn.

Minimum budget = `Σ(orderMinSize × p_i)` for each selected
leg — not `orderMinSize × leg_count`. If budget is below
this sum, report the constraint and stop.

**Zero-liquidity check:** if any selected bin shows
`yesPrice` of 0 or near-zero spread in discovery data,
flag it as likely untradeable before presenting the plan.
Do not include zero-liquidity bins in the plan without
warning.

## 4. explain

Present:
- shape name
- actual covered bins
- per-leg size and cost
- totals
- `min order: {orderMinSize} shares per leg`
- a **scenario table**

**EVERY bin gets a row.** The scenario table MUST include
one row per actual market bin — ALL of them, not just the
selected ones. Show unselected bins as zero rows. This is
non-negotiable.

Example — corridor 68-74k, size 10, **all 11 bins shown**:

```txt
bin            size   cost    payout   pnl
<64k           —      —       $0       −$8.95
64-66k         —      —       $0       −$8.95
66-68k         —      —       $0       −$8.95
68-70k         10     $3.90   $10      +$1.05
70-72k         10     $4.10   $10      +$1.05
72-74k         10     $0.95   $10      +$1.05
74-76k         —      —       $0       −$8.95
76-78k         —      —       $0       −$8.95
78-80k         —      —       $0       −$8.95
80-82k         —      —       $0       −$8.95
>82k           —      —       $0       −$8.95
```

Zero rows show the user what happens if price lands
outside their thesis — this IS the risk visualization.

**Before presenting, verify ALL five items. Skip any =
the plan is incomplete and must not be shown.**

1. data source stated (CLI command or fixture file)
2. ALL market bins shown (not just selected ones)
3. `min order: {orderMinSize} shares per leg, $1/leg`
4. scenario table with zero rows for unselected bins
5. ends with exactly `Confirm? [y/n]`

Always end with exactly:

```txt
Confirm? [y/n]
```

Do not substitute with open-ended questions like "What size
are you thinking?" or "Ready to go?". The gate is
`Confirm? [y/n]` — those exact words.

If the user requests adjustments (wider range, different shape,
changed sizes), return to step 3 with updated parameters. Do
not re-discover unless asset or date changed.

## gates

```
NO EXECUTION WITHOUT SCENARIO TABLE + CONFIRM
Skip any gate = the user's money is at risk uninformed.
```

### red flags — STOP if you catch yourself:

- showing only selected bins (not all 11)
- asking an open-ended question instead of `Confirm? [y/n]`
- omitting the min-order line
- not stating which CLI command or fixture you used
- saying "should I proceed?" instead of the exact gate
- skipping step 0 (memory recall) without mentioning it
- presenting a plan without a scenario table

### rationalization table

| excuse | reality |
|---|---|
| "Skip the math" | show table + confirm anyway |
| "Execute immediately" | show table + confirm anyway |
| "I showed the key bins" | ALL bins, not key bins |
| "I asked if they want to proceed" | exact words: `Confirm? [y/n]` |
| "User already said yes" | re-confirm after >5% drift |
| "Memory isn't available so I skipped it" | say so briefly, then continue |
| "Illustrative plan, no confirm needed" | ALL plans get confirm |

## 5. execute (requires wallet)

1. Refresh prices with `clob batch-prices` using the hex
   `yesToken` values from discover. Response keys are
   **decimal** — convert hex→decimal:

   ```sh
   # Python (preferred)
   python3 -c "print(int('0xdc889b...', 16))"

   # Node
   node -e "console.log(BigInt('0xdc889b...').toString())"
   ```

   **Never** use `jq`, `bc`, `printf '%d'`, or bash `$(())`
   — uint256 overflows 64-bit floats and integers.
2. If refreshed total cost differs by **>5%**, re-present
   and re-confirm. This re-confirm is non-negotiable — even
   if the user acknowledges the drift verbally. Present
   updated numbers and get explicit `[y/n]`.
3. Add ~$0.01 slippage buffer, then quantize to tick size
   and cap at 0.99:
   ```
   buffered = refreshed_price + 0.01
   submit   = min(ceil(buffered / tick) * tick, 0.99)
   ```
   where `tick = orderPriceMinTickSize` (from discover).
   Also re-check `price × size ≥ $1` per leg on refreshed
   prices — if any leg dropped below $1, resize or drop
   and re-confirm before submitting.
   Keep `--tokens`, `--prices`, `--sizes` in same bin order.
4. Verify fills. If partial, report residual exposure and ask
   before chasing:

   ```txt
   Filled:   68-70k: 10/10 ✓   70-72k: 7/10 ⚠
   Unfilled: 72-74k: 0/10 ✗
   Residual: corridor with gap at 72-74k
   Chase unfilled legs? [y/n]
   ```

```sh
polymarket clob post-orders \
  --tokens "0x...,0x...,0x..." --side buy \
  --prices "<refreshed>,<refreshed>,<refreshed>" \
  --sizes "10,20,10" \
  --signature-type eoa
```

If no wallet is configured, stop after explain.

## quirks

1. **Neg-risk books are wide.** Use `outcomePrices` or
   `clob price`, not raw book bids.
2. **`events` only on slug fetch.** Use slug, not numeric ID.
3. **Always `--signature-type eoa`.** Proxy mode is broken.
4. **Collateral is USDC.e**, not native USDC.
5. **Limit orders only.** PM CLOB has no market-order type.
   Use refreshed price + ~$0.01 buffer. If the user asks
   for market orders, explain and use the standard limit
   approach.

## out of scope

- binary (non-bracket) markets
- threshold/strike markets (e.g., "BTC above X")
- touch-price / intraday resolution
- volatility surfaces or implied vol trades
- portfolio-level allocation across multiple assets
- wallet setup, API credentials, or account funding
- options strategies that require selling (PM = buy only)

If the user asks for any of the above, say so and stop.

## fixture fallback

If `polymarket` CLI is unavailable, read fixture JSON from
the `fixture/` directory adjacent to this SKILL file.
Same structure as `events get` output.

Fixtures are point-in-time snapshots. If no fixture matches
the resolved date, say: **"Offline mode only covers
{available dates}. Connect the CLI for live data."**
Do not use a mismatched fixture without warning the user
that prices and bins are from a different date.
