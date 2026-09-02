# Marketplace under catalog scarcity: a knob that does nothing

MAS.665 AI Studio onboarding experiment (Step 2). One scenario, one setting changed,
one hypothesis stated before running.

## The choice

Scenario: **marketplace** (100 agents, 50 buyers / 50 sellers, seed 42, 10,000 ticks).

Setting changed: `task.config.catalog_size`, from **200 to 20**. Nothing else.

I picked an economic knob instead of a failure knob on purpose. `failures.message_drop`
is the example the quickstart already walks you through, so its outcome is known before
you run it. `catalog_size` looked like a supply-side lever: with a 10x smaller catalog,
50 buyers should be competing for far fewer goods.

## The hypothesis (written before running)

- `deal_rate` falls from the 0.532 baseline and `rejection_rate` rises, because there
  is less to go around.
- `mean_rounds_to_deal` rises above 1.02, because buyers contend for the same items.
- `unique_pairs` falls below 467, because trade concentrates on the sellers who still
  hold inventory.
- `marketplace_no_double_sell` stays PASS, but over more concurrent attempts on the
  same item. A FAIL here would have been the interesting result: a race in coordination
  under contention.

## What actually happened

Nothing. Not "a small effect", literally nothing:

```
$ shasum -a 256 traces/marketplace.jsonl traces/marketplace-catalog20.jsonl
54c0870fbf7c2700f1dd73210131444d20066b47b091ae18c987d7773cf56e9d  traces/marketplace.jsonl
54c0870fbf7c2700f1dd73210131444d20066b47b091ae18c987d7773cf56e9d  traces/marketplace-catalog20.jsonl
```

The two traces are byte-for-byte identical. Every metric matches to four decimals.

| Metric | Baseline (catalog_size 200) | Changed (catalog_size 20) |
|---|---|---|
| deal_rate | 0.5320 | 0.5320 |
| rejection_rate | 0.4680 | 0.4680 |
| mean_rounds_to_deal | 1.0226 | 1.0226 |
| unique_pairs | 467 | 467 |
| message_count | 2000 | 2000 |
| delivery_rate | 1.0000 | 1.0000 |

Property validators, identical on both traces:

```
PASS marketplace_no_double_sell   checked 266 sales
PASS marketplace_all_responded    all 500 requests answered
PASS marketplace_price_agreement
```

## How I investigated it

My first assumption was my own mistake: a typo in the YAML, or `nest run` silently
falling back to the built-in scenario. The scenario name in the run banner and the new
trace path ruled that out, and `diff` against `scenarios/marketplace.yaml` confirmed
`catalog_size` was the only changed line.

So I went to the source:

```
$ grep -rn "catalog_size" packages/ scenarios/
packages/nest-core/nest_core/scenario.py:92:  task = TaskConfig(type="marketplace", config={"catalog_size": 200})
packages/nest-core/nest_core/scenarios_builtin/yaml/marketplace.yaml:35:    catalog_size: 200
scenarios/marketplace.yaml:35:    catalog_size: 200
```

Three hits, all of them declarations. Not one read. `marketplace_factory` in
`packages/nest-core/nest_core/scenarios_builtin/marketplace.py:336` pulls exactly one key out of `task.config`:

```python
task_config = config.task.config
rounds = task_config.get("rounds", 10)
```

`catalog_size` is a documented knob (`docs/writing-a-scenario.md:55`) that is never
wired to anything. There is no catalog and no inventory in the marketplace task at all:
sellers decide with a `min_price` threshold, so scarcity has nothing to act on.

**Control run.** A null result is only evidence if the setup can detect a non-null one.
I re-ran the same file changing `rounds` from 10 to 3, the one key the factory does
read, and the pipeline moved as expected: `message_count` 2000 to 600, `unique_pairs`
467 to 143, `deal_rate` 0.5320 to 0.5067. The comparison harness is sensitive. The knob
is inert.

**Scope check.** I scanned every `task.config` key across `scenarios/*.yaml` against the
scenario sources. `catalog_size` is the only one with no reader. `network_partition.groups`
looked inert by the same test but turned out to be read in `packages/nest-core/nest_core/runner.py:159`, one layer up
from the scenario factories, so it is a false positive from my grep, not a second bug.

## What I take from this

`task.config` is a free-form dict with no schema, which the docs say plainly. The
consequence is not obvious until it bites you: an unread key fails silently, so a
scenario can look configured while nothing downstream is listening. In a
multi-agent benchmark that is the worst failure mode available, because it produces
confident numbers instead of an error. The fix is cheap: reject unknown keys in
`task.config`, or warn on them.

It also changed how I read a result. My first instinct at "identical output" was to
distrust my own setup, which was right, but the way out was the control run, not more
staring at the YAML. Determinism made the diagnosis trivial: with a fixed seed, an
identical hash is not weak evidence of no effect, it is proof.

## Reproducing

```bash
nest run scenarios/marketplace.yaml
nest run examples/marketplace-catalog-scarcity/scenario.yaml
shasum -a 256 traces/marketplace.jsonl traces/marketplace-catalog20.jsonl
```

## Tools and help

Claude Code (Opus) for repo setup (fork, Python 3.12 venv, `nest-core[plugins]`), running
both scenarios, diffing reports, running the validators, and drafting this README. I chose
the scenario and the setting, wrote the hypothesis before running, designed the control run,
and read the trace evidence and the source myself. Docs used: `docs/quickstart.md` and
`docs/writing-a-scenario.md`. No other human help.
