---
layout: post
title: "Your Eval Suite Is Not Your Release Gate"
subtitle: "A hundred passing examples are consistent with a 3% failure rate. Here is the arithmetic, and what to put in CI instead of a score."
---

A user reports that dates on one supplier's invoices come back a month off, because that supplier writes them day-first. Someone adds a line to the prompt clarifying the date format, and the bug goes away. What also happens, quietly, is that the model now reads the line-item table header differently, so on documents where the currency appears only in that header, the currency field starts coming back null.

Nobody notices, because the overall score moved from 94.1% to 94.0%, and that looks like noise.

That team was not careless. They ran the evals. Comparing two aggregate scores, on a set anyone can afford to label, cannot detect the regression they cared about. The arithmetic says so before you write a line of harness code.

## Two scores cannot detect anything

Say your candidate passes all 100 hand-labelled examples. The rule of three says that with zero failures in *n* trials, the 95% upper bound on the true failure rate is about 3/*n*. A clean run on 100 examples is consistent with a true failure rate of 3%. At a thousand documents a week, that is thirty broken ones your green build considers impossible.

That is the optimistic reading. The rule of three assumes those 100 were drawn at random from the population you care about, and a golden set is curated, skewed toward cases you already know about. So it says even less about inputs nobody has thought of yet.

Comparing two versions is worse. Sizing a difference between two proportions, at 5% significance and 80% power, takes roughly

```
n ≈ 8 · [ p₁(1−p₁) + p₂(1−p₂) ] / δ²     per arm
```

To detect a drop from 94% to 92% (illustrative; put your own numbers in) that is around 2,500 examples per arm. Nobody labels 2,500 examples to approve a prompt tweak. So teams run two hundred examples twice, see two scores a tenth of a point apart, and conclude nothing changed. They were right that they saw nothing, and wrong to read that as nothing having happened.

## Pairing is where the power actually lives

You are not running a trial on two groups of strangers. You run both versions over the same items, which makes the comparison paired. In a paired comparison, the items both versions get right and the items both get wrong carry no information. Only the disagreements do.

So stop comparing scores and count transitions: items the baseline got right and the candidate now gets wrong, and the reverse. Under the null hypothesis that your change is neutral, each disagreement is a coin flip. That is McNemar's test, and its power depends only on how many discordant pairs you have, not on how big your set is.

Which lets you read your own detection limit off a table:

| discordant pairs | minimum split to reach p < 0.05 |
|---|---|
| 5 | unreachable at any split |
| 6 | 6-0 |
| 10 | 9-1 |
| 12 | 10-2 |
| 20 | 15-5 |
| 30 | 21-9 |
| 50 | 33-17 |

*(two-sided exact binomial, p = 0.5; derive your own rows in three lines of Python)*

Read that as: a change is not statistically visible until the disagreements run roughly three to one. If your candidate and baseline agree on 97% of a 600-item set (illustrative; measure yours) you get about 18 disagreements and need 14 pointing the same way. Anything more balanced is, formally, nothing.

## The suite measures. The gate asserts.

The arithmetic points somewhere counterintuitive: do not make the statistics your gate. As a merge criterion a significance test fails to fire when you need it and is impossible to explain when it does. "Your PR is blocked because 11 of 14 discordant pairs went the wrong way" is not a conversation that ends well on a Friday evening.

Your diagnostic set is a measuring instrument, and it should be hard. The most-amplified advice here is Hamel Husain's, that a 100% pass rate means your evals are too easy and something nearer 70% is more informative. That is good advice for an instrument and unusable as a merge criterion, because you cannot block a deploy on a number deliberately engineered to sit at 70%, and you cannot tell 69% from 71% at any sample size you will pay for.

Your release gate is not an instrument. It is an assertion, and it should be boring. The items on it are ones you have decided must keep working, it sits near 100% by construction, and the rule is a rule rather than a statistic: zero newly-broken items on the fields and slices you have designated critical, and a written explanation for anything newly broken elsewhere. A slice is a subpopulation you can name in advance, such as document length, language, layout family, or whether the field is present at all.

That gives you two objects, with two samples and two thresholds. Most teams have one and ask it to do both jobs.

## Your threshold comes from your noise floor

A zero-regression rule only works if your pipeline is repeatable, and it probably is not. Temperature zero is not a determinism switch on hosted inference: batch composition varies, floating-point addition is not associative, and identical requests can return different outputs.

Measure it before you set anything. Run the identical pipeline twice over the same set and diff the two runs against each other rather than against the labels. The fraction of items that disagree is your noise floor, and running the whole set twice gives one paired comparison per item, which is the cheapest design available.

If you see zero disagreements, do not conclude the pipeline is deterministic. Apply your own rule of three: zero across N comparisons puts the upper bound near 3/N, so 600 clean comparisons still allows a per-item flip rate around half a percent, which is roughly one free failure per run on a 200-item gate.

That is the contradiction inside every zero-regression rule, and it is why gates get switched off. The fix is mechanical: confirm before you fail. When an item flips, re-run only that item a few times. Fail on items that flip consistently, log the rest as flakes, and track the flake rate as a number in its own right. You only pay for items that already moved, and it is the difference between a gate people trust and one people re-run until it goes green.

Do not score structured output with an LLM judge. That puts a second drifting model inside the mechanism meant to detect drift in the first. Wanting a judge there usually means you have a normalisation problem rather than a scoring problem.

## Where it runs

Run it pre-merge on any diff touching prompts, schemas, parsing or model configuration. This has to be fast, so run a subset if the full set is expensive, and make that subset fixed. A randomly sampled one makes the gate itself non-deterministic, which is the opposite of what you are building. Post-merge and pre-deploy, run everything, including the slow and expensive cases.

Then run it on a schedule, against the live provider, with nothing changed on your side. This is the one everyone skips. A version pin is a promise about the weights. It is not a promise about the serving stack around them: default sampling parameters, how your SDK serialises tool schemas, server-side truncation, safety filtering. Before you file a provider incident, check your own lockfile. More often than you would expect, "the model changed" turns out to be a library upgrade that altered how the prompt gets assembled. A gate that only runs when you push assumes you are the only source of change, and you are not.

## How gates die: golden-set capture

The way a gate actually dies is that someone repairs the golden set.

Build goes red. Engineer looks at the three failing items, decides the model's new answers are arguably fine, edits the expected values, build goes green. Do that four times and the set no longer specifies correct output; it describes what your system does. Call it *golden-set capture*: the thing being regulated has quietly taken over the regulator. After that it detects nothing, while still costing a full run on every pull request.

There is a second, less visible route to the same place. The affordable way to build a set is to run your pipeline over real inputs and have someone who knows the domain *correct* the outputs rather than write them from scratch. Do this; the alternative is having no set at all. But a human shown an answer agrees with it far more readily than one asked to produce it, so corrected labels are anchored, by construction, to the behaviour of the system they exist to audit.

The defences are cheap:

- Changes to expected values go in a separate commit from changes to code, reviewed by someone who is not shipping the feature.
- Never change the scorer in the same pull request as the prompt. A quiet widening of the comparator absorbs a real regression and leaves nothing visible in the diff.
- Blind-label a subsample: hide the model output, have someone answer from scratch, and measure how far those labels sit from the corrected ones. That delta is the size of your anchoring problem. Build the critical slice from scratch.
- Give the gate an override that is logged, attributed and expiring. A gate with no escape hatch gets deleted the first time it blocks a release, and one whose overrides never expire is a test you have already deleted without noticing.

## Where to start

Build this the moment output flows into another system with no human in between, or when more than one person can change the prompt. Earlier than that, while the output shape still changes weekly, your expectations go stale before they are useful.

If you are starting this week, the useful order is not the one teams use. Measure your noise floor. Pick the three or four fields where a wrong value is expensive. Write down from scratch, without looking at model output, the expected result for a dozen items: your cleanest input, your ugliest, and the last bug a user reported. Gate on zero newly-broken items among those, with confirm-before-fail. That fits in an afternoon and it is a real gate.

The harness is the easy part, which is exactly why teams start there.
