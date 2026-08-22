# A 2% fault removes most of the market, and the instruments do not report it

**Scenario:** [`scenarios/marketplace_byzantine.yaml`](../../../scenarios/marketplace_byzantine.yaml)
**One setting changed from `scenarios/marketplace.yaml`:** `failures.byzantine_agents: 0.0` to `0.02`
**Held fixed:** 50 buyers, 50 sellers, all 12 layer plugins, `rounds: 10`, 10000 ticks.
**Seeds:** the committed scenario is seed 42. Every claim below is also checked on seeds 7, 1337, 2026, 99.

## Part A: the baseline

`nest run marketplace`. 100 agents, 2000 messages, 500 buy requests, 266 sales,
`deal_rate` 0.532, `delivery_rate` 1.000, and all 50 buyers complete all 10 of
their rounds. Reruns are byte-identical, which is what makes a one-knob
comparison worth anything.

## Part B: the change, and why

I picked `failures.byzantine_agents` over the `message_drop` knob the quickstart
suggests, because of something in `nest_core/scenarios_builtin/marketplace.py`
that I read before running anything.

The buyer is a strict request/response state machine with **no timeout and no
retry**. It advances its round counter only inside `on_message`, and
`_verify_response` returns `(msg, False)` on a signature failure, at which point
`on_message` returns without sending. So a buyer that receives a single
unverifiable message does not retry, reports nothing, and never sends again. It
is retired for the rest of the run.

Byzantine mode XORs random bytes into everything a faulty agent sends
(`sim/simulator.py`, `deliver` branch). Garbled bytes fail verification. So
Byzantine faults should not corrupt one transaction, they should permanently
remove healthy counterparties from the market.

**Hypothesis:** at a 2% Byzantine fraction, market activity falls far more than
2%, and `deal_rate` falls with it.

The first clause held. The second was wrong, and being wrong about it is the
result.

## What happened

Seed 42, the committed scenario:

| run | byz | deal_rate | delivery_rate | buy requests | buyers finishing all 10 rounds |
|---|---|---|---|---|---|
| baseline | 0.00 | 0.532 | 1.000 | 500 | 50 / 50 |
| **submitted** | **0.02** | **0.587** | **1.000** | **390** | **30 / 50** |
| | 0.10 | 0.608 | 1.000 | 273 | 14 / 50 |
| | 0.40 | 0.585 | 1.000 | 94 | 0 / 50 |

Two faulty agents out of one hundred permanently stalled 20 of 50 buyers and
erased 22% of market activity. The amplification was real and bigger than I
expected.

But `deal_rate` went **up**. `delivery_rate` never left 1.000. So the finding is
not "Byzantine agents hurt the marketplace." It is that **the damage does not
appear in the instruments, and the headline metric moves in the direction that
reads as improvement.**

### Across five seeds

| seed | deal_rate 0.00 → 0.02 | buyers lost @0.02 | deal_rate @0.10 | buyers lost @0.10 |
|---|---|---|---|---|
| 42 | 0.532 → 0.587 | 20 | 0.608 | 36 |
| 7 | 0.516 → 0.543 | 16 | 0.610 | 39 |
| 1337 | 0.556 → 0.576 | 11 | 0.620 | 31 |
| 2026 | 0.520 → 0.541 | 11 | 0.587 | 30 |
| 99 | 0.546 → 0.544 | 2 | 0.597 | 30 |

What survives the seed check:

- `delivery_rate` is **1.000 in all 15 runs**. It never registers anything.
- `deal_rate` **never falls** in response to the fault, 5 seeds out of 5. It
  rises in 4 of 5 at 0.02 and 5 of 5 at 0.10.
- At `byz: 0.10`, **30 to 39 of 50 buyers** are permanently stalled, in every
  seed.

What does not survive: the size of the effect at 0.02. Seed 99 lost only 2
buyers, because only one of its two faulty agents was a seller. At this fraction
the sample is two agents, so which two matters enormously. The direction is
stable; the magnitude at 0.02 is not, and I would not quote "20 of 50" as an
expectation.

## Investigation

### 1. Why `delivery_rate` is blind

Corruption is not loss. `_should_drop` runs before the Byzantine XOR, so garbled
messages are counted as delivered and `dropped_count` stays 0. As a control I ran
the same sweep on `message_drop` instead: at 2%, `delivery_rate` falls to 0.979
with 17 drops. The instrument works. It measures arrival, and these messages
arrive. They arrive as noise.

### 2. Why `deal_rate` rises

`deal_rate` is sales over buy requests, so it is conditioned on requests that
happened. It cannot see requests that were never made.

My first guess was product contention: sellers keep a `_sold_products` set and
reject any repeat of a product ID they already sold, so fewer surviving buyers
means less collision on `product-0` through `product-9`. To test that instead of
assuming it, I ran a **fault-free** control varying only the buyer count:

| buyers, no faults | 50 | 40 | 30 | 20 | 10 |
|---|---|---|---|---|---|
| deal_rate | 0.532 | 0.578 | 0.637 | 0.680 | 0.760 |

In this scenario `deal_rate` is a monotonically decreasing function of
participation, with no faults present at all. The Byzantine "improvement" is the
shape of a smaller market, not a healthier one. The 0.10 run, whose surviving
traffic is equivalent to roughly 27 active buyers, lands at 0.608, next to the
fault-free 30-buyer control at 0.637.

### 3. Why the trust layer never learns

The scenario runs `trust: score_average`, and sellers do refuse buyers scoring
below 0.2. But evidence is only recorded on a verified `sold:` (positive) or a
verified `reject:` (negative). A garbled message is neither. Verification fails,
the handler returns, and **no evidence of any kind is written**.

Reconstructing the evidence stream from the trace (a receive whose body parses as
none of `buy:`/`sold:`/`reject:` was corrupted in flight, so its sender is
Byzantine):

```
seller       positive negative  no-evidence  byzantine
seller-26           0        0            9   YES
seller-3            0        0           11   YES
seller-0            6        2            0   no
seller-1            6        6            0   no

across all Byzantine sellers at byz=0.10, seed 42: positive=0, negative=0, no-evidence=35
```

Every Byzantine seller ends the run with a completely empty reputation record
while honest sellers accumulate both praise and complaints. The reputation
system's only sensor is a well-formed complaint, so the agents behaving worst are
the ones it knows least about. A merchant that answers with noise is not
low-reputation, it is **unrated**, and unrated is treated as fine.

### 4. What the validators actually detect

There are two silent death modes. A healthy buyer that picks a Byzantine seller
dies on the garbled reply. A Byzantine buyer dies because its own garbled request
fails the seller's verification and the seller returns without answering.

Only the second is visible. `marketplace_all_responded` counts unanswered
requests, and across all ten fault runs that count is **exactly the number of
Byzantine buyers**, every time:

| seed | byz | Byzantine buyers | validator reports | buyers actually stalled |
|---|---|---|---|---|
| 42 | 0.02 | 0 | all pass | 20 |
| 7 | 0.02 | 0 | all pass | 16 |
| 1337 | 0.02 | 1 | 1 unanswered | 11 |
| 2026 | 0.02 | 1 | 1 unanswered | 11 |
| 99 | 0.02 | 1 | 1 unanswered | 2 |
| 42 | 0.10 | 3 | 3 unanswered | 36 |
| 7 | 0.10 | 3 | 3 unanswered | 39 |
| 1337 | 0.10 | 6 | 6 unanswered | 31 |
| 2026 | 0.10 | 5 | 5 unanswered | 30 |
| 99 | 0.10 | 5 | 5 unanswered | 30 |

The validator is not broken, it is answering a narrower question than a reader
will assume. Where it reports a number, that number understates the stalled
population by 5x to 13x. On the two seeds where both faulty agents happened to be
sellers, all three validators pass cleanly while 40% and 32% of the buyer
population is dead.

## What I take from this

Rate metrics conditioned on completed work cannot detect participants who stopped
participating, and in an agent marketplace that is the failure that matters most,
because it is the one that looks like success. If I were instrumenting a real
agent marketplace, the first metric I would add is not a success rate. It is a
count of counterparties still transacting at the end of the window against the
start.

Two narrower lessons. A request/response agent with no timeout converts any
integrity failure into a permanent availability failure. And a reputation system
that learns only from well-formed messages is structurally unable to see the
agents that send malformed ones.

## Limits, stated plainly

- Five seeds, one configuration. The direction of every claim above is stable
  across seeds; the magnitude at `byz: 0.02` is not, and ranges from 2 to 20
  buyers lost depending on whether the two faulty agents land on the seller side.
- I did not fix anything. Adding a buyer timeout, and reporting negative evidence
  on a verification failure, are the two obvious next arms and I have not run
  them.
- The corrupted-receive heuristic for identifying Byzantine senders would
  mislabel a message whose XOR mask drew all zeros. That needs every byte to draw
  0, so I treated it as safe.
- Tier 1 state machines only. An LLM-backed agent might well retry on garbage,
  and nothing here says what happens then.

## Reproduce

From a clone of this repo (Python 3.12+):

```bash
uv sync
uv run nest run scenarios/marketplace.yaml
uv run nest run scenarios/marketplace_byzantine.yaml

uv run python - <<'PY'
from pathlib import Path
from nest_core.metrics import compute_metrics
from nest_core.validators import validate_trace
for n in ("marketplace", "marketplace_byzantine"):
    p = Path(f"traces/{n}.jsonl")
    print(n, compute_metrics(p, ["deal_rate", "delivery_rate", "message_count", "dropped_count"]))
    for r in validate_trace(p, "marketplace"):
        print("   ", "PASS" if r.passed else "FAIL", r.name, "-", r.detail)
PY
```

Expected for `marketplace_byzantine`: `deal_rate` 0.5872, `delivery_rate` 1.000,
1560 messages, 0 dropped, all three validators passing.

The buyer-survival and trust-evidence tables come from this, over either trace:

```bash
uv run python - <<'PY'
import json
from collections import Counter, defaultdict
from pathlib import Path

KNOWN = ("buy:", "sold:", "reject:")
events = [json.loads(x) for x in Path("traces/marketplace_byzantine.jsonl").read_text().splitlines()]
body = lambda e: str(e.get("msg", "")).rsplit("|sig:", 1)[0]

buys, trust, blind, byz = Counter(), defaultdict(lambda: [0, 0]), Counter(), set()
for e in events:
    a, f, k = str(e.get("agent", "")), str(e.get("from", "")), e.get("kind")
    if k == "send" and body(e).startswith("buy:"):
        buys[a] += 1
    if k == "receive":
        b = body(e)
        if not b.startswith(KNOWN):
            byz.add(f)
            if a.startswith("buyer-") and f.startswith("seller-"):
                blind[f] += 1
        elif a.startswith("buyer-") and f.startswith("seller-"):
            trust[f][0 if b.startswith("sold:") else 1] += 1

print(f"buyers finishing all 10 rounds: {sum(1 for a in buys if buys[a] >= 10)} / {len(buys)}")
print(f"byzantine senders seen: {sorted(byz)}")
for s in sorted(byz):
    if s.startswith("seller-"):
        print(f"  {s:<12} positive={trust[s][0]:<3} negative={trust[s][1]:<3} "
              f"no-evidence={blind[s]:<3}")
PY
```

For the seed table, copy `scenarios/marketplace_byzantine.yaml`, change `seed:`
to 7, 1337, 2026, or 99, point `output.trace` somewhere new, and rerun the two
blocks above.

## Use of AI and other help

Written with Claude (Opus) in an agentic session, which is also the subject
matter of this course, so hiding it would be the wrong move.

- **Mine:** the choice of knob, the decision to read the agent source before
  running anything, the hypothesis, and the three follow-ups that turned an odd
  number into an explanation (the fault-free buyer-count control, reconstructing
  the trust evidence stream from the trace, and the multi-seed check that forced
  me to weaken the validator claim below).
- **Claude's:** running the sweeps, writing the throwaway analysis scripts, and
  drafting this README from the numbers.
- **What changed under checking:** my first draft said "all three validators
  pass," on the strength of seed 42 alone. Running five seeds showed that holds
  on 2 of 5, and led to the sharper and more useful claim in section 4, that the
  validator's count equals the Byzantine buyer count exactly and misses all
  seller-side damage.
- Every figure here was re-run from the committed YAML. `nest doctor` reports
  7/7. Reruns are byte-identical. No sources beyond this repository, read at
  `nest-core` 0.1.4.
